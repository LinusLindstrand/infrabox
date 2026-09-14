<div id="toc">
  <ul style="list-style: none">
    <summary>
      <h1> Kea DHCP4 High-Availability Setup — Runbook </h1>
    </summary>
  </ul>
</div>

## Overview

Two Kea DHCP4 servers behind an OPNsense relay, running in **hot-standby** HA mode:

| Role      | Host          | IP           |
|-----------|---------------|--------------|
| Primary   | kea-primary   | 10.20.2.41   |
| Secondary | kea-secondary | 10.20.2.42   |

Each server is hosted as an LXC on proxmox and both servers listen on interface `eth0` and receive DHCP requests **relayed by OPNsense** if not on the same broadcast domain. OPNsense forwards client DISCOVER/REQUEST packets as unicast to both Kea servers.

Subnets currently configured:
- **VLAN 101** (`10.10.1.0/24`) — priv/devices, relay `10.10.1.1`, DNS `10.20.2.223`
- **VLAN 201** (`10.20.1.0/24`) — prod/mgmt, relay `10.20.1.1`, DNS `10.20.2.223`
- **VLAN 202** (`10.20.2.0/24`) — prod/infra, relay `Null`, DNS `10.20.2.223`
- **VLAN 205** (`10.20.5.0/24`) — prod/services, relay `10.20.5.1`, DNS `10.20.2.223`

--- 

## Config file structure (`/etc/kea/kea-dhcp4.conf`)

Top-level sections, in the order they appear: 

1. `valid-lifetime` / `renew-timer` / `rebind-timer` — lease timing (seconds), standard ratio: renew = 1/2 valid, rebind = 7/8 valid
2. `interfaces-config.interfaces` — list of interfaces Kea listens on (`eth0` in this setup).
3. `lease-database` — memfile backend, path `/var/lib/kea/kea-leases4.csv`.
4. `control-socket` — unix socket for live queries/commands (`/run/kea/kea4-ctrl-socket`).
5. `multi-threading` — `enable-multi-threading: true` required for the HA hook's HTTP listener to function properly.
6. `hooks-libraries` — **order matters**: `libdhcp_lease_cmd.so` must load *before* `libdhcp_ha.so`, since HA sync depends on lease commands (`lease4-get-page`, etc.) that only exist once `lease-cmds` is loaded. Parameters for HA — mode: `hot-standby`, heartbeat-delay, max-reponse-delay, max-ack-delay, and max-unacked-clients. Parameters for HA peers — name, url, and role of the peer.
7. `subnet4` — array of subnet blocks. **Must be indetical on both HA peers.**

---

## Adding a new subnet

Copy this template into the `subnet4` array. Will be automated by ansible in the future.

```json
{
  "id": <unique-integer>,
  "subnet": "<network>/<cidr>",
  "comment": "<description>",
  "interface": "eth0",
  "relay": {
    "ip-addresses": [ "<opnsense-gateway-ip-for-this-vlan>" ]
  },
  "pools": [ { "pool": "<start> - <end>" } ],
  "option-data": [
    { "name": "routers", "data": "<gateway-ip>" },
    { "name": "domain-name-servers", "data": "<dns-ip>" }
  ]
}
```

Notes:
- `id` must be globally unique across the config. Convention used here: match it to the VLAN number.
- `relay.ip-addresses` takes an **array** (plural key) — `"ip-address"` (singular) is invalid syntax in Kea 2.6.
- Keep pool range narrower than the full subnet to leave room for static reservations.
- **Apply the same block to both server1 and server2 configs** — HA does not sync subnet definitions, only lease state.
- OPNsense's relay config for that VLAN interface must forward to **both** `10.20.2.41` and `10.20.2.42`.

---

## Adding a static reservation

Add a `reservation` array as a sibling key inside the relevant `subnet4` block:

```json
"reservations": [
  {
    "hw-address": "aa:bb:cc:dd:ee:ff",
    "ip-address": "10.x.x.x",
    "hostname": "some-host"
  }
]
```

- `hw-address`: lowercase, colon-separated MAC.
- `ip-address`: pick something **outside** the dynamic pool range to avoid any race with pool allocation.
- Apply to both server configs (same reasoning as subnets — not synced by HA).

---

## HA hook configuration

Both servers need identical `hooks-libraries` entries except for `this-server-name`:

```json
"hooks-libraries": [
  {
    "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_lease_cmds.so"
  },
  {
    "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_ha.so",
    "parameters": {
      "high-availability": [
        {
          "this-server-name": "server1",   // "server2" on the secondary
          "mode": "hot-standby",
          "heartbeat-delay": 10000,
          "max-response-delay": 60000,
          "max-ack-delay": 5000,
          "max-unacked-clients": 5,
          "peers": [
            { "name": "server1", "url": "http://10.20.2.41:8000/", "role": "primary" },
            { "name": "server2", "url": "http://10.20.2.42:8000/", "role": "standby" }
          ],
          "http-dedicated-listener": true
        }
      ]
    }
  }
]
```

Key gotchas hit during setup:
- `http-dedicated-listener` goes **inside** the `high-availability` object, not in the top-level `multi-threading` section (Kea will reject it there with a syntax error).
- The HA hook opens its own HTTP listener directly inside `kea-dhcp4` on the port given in each peer's own `url` — no separate control-agent process is needed.
- Port `8000` (or whatever you choose) must be reachable **both directions** between the two hosts.

---

## Critical fix: systemd boot-race (interface not up yet)

**Symptom:** Kea fails to start after a reboot with:
```
DHCPSRV_OPEN_SOCKET_FAIL failed to open socket: the interface eth0 is down
...
CmdHttpListener::run failed: ... bind: Cannot assign requested address
```

**Cause:** `kea-dhcp4-server.service` starts before `eth0` has an IP, because `network-online.target` isn't actually gated on real interface readiness — `systemd-networkd-wait-online.service` was disabled.

**Fix (apply on both hosts):**

```bash
sudo systemctl enable systemd-networkd-wait-online.service

sudo mkdir -p /etc/systemd/system/systemd-networkd-wait-online.service.d/
sudo tee /etc/systemd/system/systemd-networkd-wait-online.service.d/override.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --interface=eth0
EOF

sudo systemctl daemon-reload
```

The Kea unit already ships with `Wants=network-online.target` / `After=network-online.target` — no override needed there. The problem was purely that nothing was backing that target with a real wait condition for `eth0` specifically.

---

## Diagnostic commands

**Validate config syntax without starting the service:**
```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

**Full untruncated service status/logs** (plain `systemctl status` truncates long lines):
```bash
sudo systemctl status kea-dhcp4-server --no-pager -l
sudo journalctl -u kea-dhcp4-server -b --no-pager | cat
```

**Check it's listening:**
```bash
sudo ss -lunp | grep 67
```

**Check HA state** (via curl over the unix control socket — no `socat` needed):
```bash
curl -s --unix-socket /run/kea/kea4-ctrl-socket \
  -X POST -H "Content-Type: application/json" \
  -d '{ "command": "ha-heartbeat" }' http://localhost/
```
Expected progression after a fresh start: `waiting` → `syncing` → `hot-standby`.

**Check HTTP reachability between peers directly:**
```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"command":"list-commands"}' http://<peer-ip>:8000/
```

**Check current leases:**
```bash
cat /var/lib/kea/kea-leases4.csv
# or live:
tail -f /var/lib/kea/kea-leases4.csv
```

**Test actual failover:**
```bash
# On primary:
sudo systemctl stop kea-dhcp4-server
# On secondary, confirm state transitions to partner-down and it starts leasing:
curl -s --unix-socket /run/kea/kea4-ctrl-socket -X POST -H "Content-Type: application/json" -d '{ "command": "ha-heartbeat" }' http://localhost/
# Then restart primary and confirm it re-syncs back to hot-standby automatically.
```

---

## JSON syntax gotchas hit repeatedly

- **Trailing comma** after the last element in an array/object — Kea rejects it (strict-ish JSON, unlike some parsers that tolerate it).
- **Missing comma** between sibling keys in the same object (e.g. between `option-data` and `reservations`).
- **Missing comma** between array elements (e.g. between two `subnet4` objects).
- Comments: `//` and `/* */` are supported as a Kea/JSON extension for hand-edited files, but are stripped and **not preserved** if the config is ever rewritten via `config-set`/`config-write`. Use the `"comment"` key instead for anything that should survive that.

# infrabox
A Proxmox cluster which will house all of my **critical** LXC applications.

## Notes & Thoughts
### The infrabox will...
- ... be clustered and use HA.
- ... be the backbone for my second project, a 19" rack.
- ... for the most part use LXCs, but exceptions can be made.
- ... use TMM PCs, likely from HP or Lenovo.
- ... strive to make the LXCs stateless.
- ... use local storage to accomplish statelessness via ceph or zfs replication.

# Inventory the existing VM lab

This provider-independent procedure inventories three Linux VMs before deployment. It does not install Ceph or configure your network. For our existing iximiuz deployment, the [finding](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) records completed checks.

1. Record all three VM identities, access method and reachable addresses. Confirm which three hosts own the dedicated disposable data devices (around 80 GiB in this lab).
2. On each VM, collect OS/kernel, CPU/RAM, root filesystem space, hostname and clock synchronization. On storage hosts, inspect block devices, mounts, filesystem signatures, partitions and LVM membership using read-only inspection such as `lsblk`, `findmnt`, `blkid`, `pvs` and `vgs` where available.
3. Identify the intended data disks by stable device identity where supported. Check device ancestry and root filesystem backing storage. A device being unmounted is insufficient proof that it is disposable. Do not wipe, partition, format, or zap devices in preflight.
4. Verify existing routed reachability among all storage nodes and from any optional clients. Check host resolution and advertised IP consistency. Record network MTU and relevant firewall restrictions; check actual Ceph listeners after bootstrap. Use your existing network; WireGuard is optional.
5. Verify host support for the selected cephadm release: Python, systemd, container runtime, time synchronization and LVM tooling. Inventory before installing prerequisites.
6. Confirm your browser can reach the intended dashboard VM on HTTPS port 8443, or plan iximiuz exposure. Native/k3s kernel and CSI checks belong to optional client extensions, not the core cluster preflight.
7. Establish whether your VM platform preserves disks, identities and IPs across stops/restarts, and whether reset/deletion destroys data. For iximiuz, include playground reset/deletion. Record resource limits and expected lab lifetime.
8. Fill the reference inventory and knowledge register. Choose pinned versions only after checking the compatibility matrix and actual resource fit.

Exit with a documented go/no-go for deployment. Unknown device ownership, insufficient memory, or missing Ceph endpoint reachability blocks bootstrap. Resource requirements must cover co-located daemons and recovery load, not just an idle cluster.

## VirtualBox or another local hypervisor

Create three Linux VMs with unique hostnames. Give each an OS disk and a separate approximately 80 GiB data disk. Use a bridged network or a host-only network that lets all three VMs and your browser host reach one another. Provide outbound connectivity for package and image downloads, for example with a separate NAT adapter. Separate default NAT adapters alone may not provide the peer connectivity this cluster needs.

Inside each VM, inspect `ip -4 -brief address` and `lsblk` to identify the reachable interface and the actual second disk. VirtualBox data disks may appear as `/dev/sdb`, whereas the iximiuz example uses `/dev/vdb`; never choose by path alone. Use the reachable VM addresses throughout the deployment and dashboard guides. Neither WireGuard nor labctl is required.

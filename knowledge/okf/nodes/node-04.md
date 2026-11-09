---
type: Virtual Machine
title: node-04
description: Optional normal VM that uses a Ceph block device. IP 192.168.56.14, 2 CPU, 6 GB memory, 20 GB OS disk, no extra disks. Defined in the Vagrantfile, not created yet.
tags: [vm, node, client]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: planned
sources:
  - id: adr-0008
    resource: ../../../docs/podman/adr/0008-rbd-clients-and-capacity-budget.md
    title: "ADR 0008: Consume replicated RBD from native and k3s clients"
---

# Facts

| Field              | Value                                                        |
|--------------------|--------------------------------------------------------------|
| Name and hostname  | `node-04`                                                       |
| CPUs and memory    | 2 CPUs, 6144 MB                                              |
| Network card 2     | `192.168.56.14` on the [private network](../networks/private-network.md) |
| Disks              | [OS disk](../storage/os-disk.md) only. Its storage comes from Ceph |
| Started by         | `vagrant up node-04` (`autostart: false`)                     |
| Created            | Not yet                                                      |
| Ansible            | Group `clients`. Listed only so the storage VMs get its `/etc/hosts` line; no play runs on it |

# Role in the lab

- Maps a 5 GiB RBD image from pool `native-rbd` (quota 6 GiB)[^adr-0008], mounts ext4 at `/srv/cephdata` and serves a file with host nginx on port 18081.
- Kernel RBD first; RBD-NBD with the pinned Ceph container only if the kernel map fails.

# Open questions

- Does kernel RBD on the box kernel 7.0.0-34 accept the 32-byte keys of Ceph 20.2.5, so RBD-NBD is not needed?

# Learn more

- [Native RBD client](../../../docs/podman/diataxis/how-to/native-rbd-client.md)

[^adr-0008]: ADR 0008: Consume replicated RBD from native and k3s clients

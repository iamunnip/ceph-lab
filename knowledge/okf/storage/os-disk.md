---
type: Disk
title: OS disk
description: Disk 1 in every VM. The box's 10 GB disk, grown to 20 GB by Vagrant. Ubuntu, the Ceph container images, logs and Prometheus data live here. Ceph never uses it as an OSD.
tags: [disk, storage, os]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Size               | 20 GB (`OS_DISK_SIZE`, `primary: true`)                    |
| Device             | `/dev/sda`, root `/dev/sda1` 19G                           |
| Grown by           | Vagrant (`Disk 'node-0N-os' needs to be resized`), then cloud-init grows the root filesystem on first boot[^finding-2026-11-08] |
| Used after boot    | 2.0G                                                       |
| Used with Ceph     | 4.7G after bootstrap; 5.2G on node-01 and node-03, 6.3G on node-02 (monitoring) on 2026-11-08 |
| Prometheus cap     | 2 GB and 15 days, to protect this disk                     |

# Rules

- Was 30 GB at first, lowered to 20 GB. VirtualBox cannot shrink a disk, so the VMs were recreated[^finding-2026-11-08].
- The data and reserve disks are also 20 GB, so size never identifies a disk. The root mount and the SCSI port do.

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

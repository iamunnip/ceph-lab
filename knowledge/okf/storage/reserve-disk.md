---
type: Disk
title: Reserve disk
description: Disk 3 in every storage VM. 20 GB, kept empty and out of Ceph until the low-space exercise, when it becomes a second OSD on each host.
tags: [disk, storage, reserve, osd]
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
| Size               | 20 GB (`RESERVE_DISK_SIZE`)                                |
| Device             | `/dev/sdc` on node-01 to node-03                           |
| SCSI port          | `/dev/disk/by-path/pci-0000:00:0f.0-scsi-0:0:2:0`          |
| State on 2026-11-08 | Blank, `Available Yes` in `ceph orch device ls`[^finding-2026-11-08] |
| Raw space          | 60 GiB without it, 120 GiB with it; about 20 and 40 GiB usable with three copies |

# Rules

- Attached from the start: Vagrant 2.4.9 deletes a disk removed from the Vagrantfile on the next reload (`VBoxManage closemedium <uuid> --delete`)[^finding-2026-11-08].
- Never use `ceph orch apply osd --all-available-devices` or the dashboard's **Add storage** button. On 2026-11-08 that button made `osd.cost_capacity` with `host_pattern: '*'` and took all three reserve disks; they were removed again with `ceph orch osd rm 3 4 5 --zap`.

# Open questions

- How long does backfill take after adding it, and how much does node-02 memory rise?
- Is 6 GB RAM enough per storage VM with two OSDs at 1.5 GiB each, plus MON, MGR and, on node-02, monitoring?

# Learn more

- [Fill the cluster and add the reserve disks](../../../docs/podman/diataxis/how-to/add-reserve-disks.md)

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

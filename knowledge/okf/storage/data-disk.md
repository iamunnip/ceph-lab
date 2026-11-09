---
type: Disk
title: Data disk
description: Disk 2 in every storage VM. 20 GB, created empty by Vagrant, then taken whole by ceph-volume as the first OSD. No manual partitions, filesystem or LVM.
tags: [disk, storage, data, osd, lvm]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: stable
sources:
  - id: adr-0004
    resource: ../../../docs/podman/adr/0004-vm-deployment-baseline.md
    title: "ADR 0004: Pin and operate the VM deployment"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Size               | 20 GB (`DATA_DISK_SIZE`)                                   |
| Vagrantfile line   | `vm.vm.disk :disk, name: "#{node[:name]}-data", size: DATA_DISK_SIZE` |
| Device             | `/dev/sdb` on node-01 to node-03                           |
| SCSI port          | `/dev/disk/by-path/pci-0000:00:0f.0-scsi-0:0:1:0`[^finding-2026-11-08] |
| Serial number      | None, so no `/dev/disk/by-id` entry                        |
| Selected by        | `paths: [/dev/sdb]` in the OSD service `osd.data-disks`    |
| After the OSD      | LVM2_member, VG `ceph-<uuid>`, LV `osd-block-<uuid>` 20G `ceph_bluestore`, PFree 0 |
| LV tags            | `ceph.osd_id`, `ceph.cluster_fsid`, `ceph.type=block`, `ceph.encrypted=0` |
| OSDs on 2026-11-08 | osd.0 on node-01, osd.1 on node-02, osd.2 on node-03, weight 0.01949 each |
| Deleted by         | `vagrant destroy`, or removing its line and running `vagrant reload` |

# Rules

- Never partitioned, formatted or wiped by hand. `ceph-volume` builds its own LVM and BlueStore on the raw disk.
- The Ansible [setup](../ansible/setup.md) role only checks it: blank, or already an LVM PV in a `ceph-...` VG with exactly one LV on this disk, whose tags name that LV and hold `ceph.type=block` and exactly this cluster's `ceph.cluster_fsid`. Anything else stops the run.

# Learn more

- [Apply the cluster layout](../../../docs/podman/diataxis/how-to/deploy-vm-cluster.md#apply-the-cluster-layout)

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

---
type: Virtual Machine
title: node-03
description: Storage VM. IP 192.168.56.13, 2 CPU, 6 GB memory, 20 GB OS disk, 20 GB data disk, 20 GB reserve disk, Ubuntu 26.04.
tags: [vm, node, storage]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: stable
sources:
  - id: adr-0009
    resource: ../../../docs/podman/adr/0009-vagrant-virtualbox-lab.md
    title: "ADR 0009: Run the VM lab on Vagrant and VirtualBox"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                        |
|--------------------|--------------------------------------------------------------|
| Name and hostname  | `node-03`                                                       |
| VirtualBox name    | `ceph-lab-podman-node-03` or `ceph-lab-snap-node-03`, by [method](../methods/index.md) |
| CPUs               | 2[^adr-0009] (needs the IO-APIC on, set in the `Vagrantfile`) |
| Memory             | 6144 MB; 5920 MiB seen by the OS[^finding-2026-11-08]        |
| OS                 | Ubuntu 26.04.1 LTS, box `cloud-image/ubuntu-26.04` 20260927.0.0 |
| Kernel             | 7.0.0-34-generic from the box, not upgraded. The 2026-11-08 cluster ran 7.0.0-43 after an upgrade that the lab no longer does |
| Network card 1     | `10.0.2.15` on the [NAT network](../networks/nat.md)       |
| Network card 2     | `192.168.56.13` on the [private network](../networks/private-network.md), VirtIO card |
| Disk 1             | [OS disk](../storage/os-disk.md) `/dev/sda`, 20 GB         |
| Disk 2             | [Data disk](../storage/data-disk.md) `/dev/sdb` `node-03-data`, 20 GB |
| Disk 3             | [Reserve disk](../storage/reserve-disk.md) `/dev/sdc` `node-03-reserve`, 20 GB |
| Started by         | `vagrant up`                                               |
| Login              | `vagrant ssh node-03`, user `vagrant`, passwordless sudo      |
| SSH key            | `vm/vagrant/<method>/.vagrant/machines/node-03/virtualbox/private_key` |
| Ansible groups     | `ceph`, `ceph_mon`, `ceph_mgr`, `ceph_osd`, `ceph_gateway` |

# Role in the cluster

- Runs a MON, a MGR, an OSD on `/dev/sdb` and a [gateway](../ceph/gateway.md).
- `vagrant reload node-03` on 2026-11-08 kept its disks, address, hostname, `/etc/hosts` lines and all Ceph daemons[^finding-2026-11-08].

# Learn more

- [VM lab reference](../../../docs/podman/diataxis/reference/vm-lab.md)

[^adr-0009]: ADR 0009: Run the VM lab on Vagrant and VirtualBox
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

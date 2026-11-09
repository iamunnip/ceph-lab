---
type: Virtual Machine
title: node-05
description: Optional single-node k3s that uses Ceph through CSI. IP 192.168.56.15, 2 CPU, 6 GB memory, 20 GB OS disk, no extra disks. Defined in the Vagrantfile, not created yet.
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
| Name and hostname  | `node-05`                                                       |
| CPUs and memory    | 2 CPUs, 6144 MB                                              |
| Network card 2     | `192.168.56.15` on the [private network](../networks/private-network.md) |
| Disks              | [OS disk](../storage/os-disk.md) only. Its storage comes from Ceph |
| Started by         | `vagrant up node-05` (`autostart: false`)                     |
| Created            | Not yet                                                      |
| Ansible            | Group `clients`. Listed only so the storage VMs get its `/etc/hosts` line; no play runs on it |

# Role in the lab

- k3s v1.36.6+k3s1, Ceph-CSI 3.18.2 with Helm v3.19.0, pool `k3s-rbd` (quota 10 GiB)[^adr-0008], an explicit RBD StorageClass, a 5 GiB RWO PVC and an nginx workload.
- Still a single VM: Ceph storage HA is not application HA.

# Learn more

- [k3s RBD client](../../../docs/podman/diataxis/how-to/k3s-rbd-client.md)

[^adr-0008]: ADR 0008: Consume replicated RBD from native and k3s clients

---
type: Ceph Pools
title: Pools
description: Replicated pools with three copies, one per host, minimum two for I/O. 60 GiB raw gives about 20 GiB usable.
tags: [ceph, pools, capacity, rbd]
status: stable
sources:
  - id: adr-0008
    resource: ../../../docs/podman/adr/0008-rbd-clients-and-capacity-budget.md
    title: "ADR 0008: Consume replicated RBD from native and k3s clients"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Pool        | State                       | Settings                                              |
|-------------|-----------------------------|-------------------------------------------------------|
| `.mgr`    | Made by the MGR             | replicated size 3, min_size 2, 1 PG                   |
| `lab-smoke` | Made on 2026-11-08        | size 3, min_size 2, pg_num 8, application `rados`[^finding-2026-11-08] |
| `native-rbd` | Planned, for [node-04](../nodes/node-04.md) | quota 6 GiB, `target_size_ratio 0.25`[^adr-0008] |
| `k3s-rbd` | Planned, for [node-05](../nodes/node-05.md) | quota 10 GiB, `target_size_ratio 0.5`[^adr-0008] |

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Failure domain     | `host`: each copy on a different storage VM               |
| Usable space       | About 20 GiB of 60 GiB raw; about 40 GiB of 120 GiB with the [reserve disks](../storage/reserve-disk.md) |
| Object placement   | `ceph osd map lab-smoke hello`: `up ([0,2,1], p0)`, three OSDs on three hosts |

# Rules

- Quotas limit use but do not reserve space. The client pools share the same physical capacity.
- Pool deletion is off by default.

# Learn more

- [Ceph HA and usable capacity](../../../docs/podman/diataxis/explanation/ha-and-capacity.md)

[^adr-0008]: ADR 0008: Consume replicated RBD from native and k3s clients
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

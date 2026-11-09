---
type: Ceph Daemons
title: Daemons
description: 3 MONs, 3 MGRs and 3 OSDs, one of each on every storage VM, placed by host label. Crash collectors on every host.
tags: [ceph, mon, mgr, osd]
resource: ../../../configs/podman/cluster-spec.example.yaml
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| MON                | node-01, node-02, node-03 (label `mon`), each on `v2:192.168.56.1x:3300` and `v1:...:6789` |
| MGR                | node-01, node-02, node-03 (label `mgr`). One active, two standby. Names on 2026-11-08: `node-01.bsllkm`, `node-02.qbaqud`, `node-03.nlbica` |
| OSD                | Service `osd.data-disks` on hosts labeled `osd`, [data disk](../storage/data-disk.md) `/dev/sdb` only, BlueStore |
| OSD memory         | `osd_memory_target` 1610612736 (1.5 GiB), `osd_memory_target_autotune false`; `MEM LIM 1536M` |
| Memory seen        | OSD 50 to 58 MiB, MGR 142 to 186 MiB, MON 26 to 45 MiB, about 5000 MiB free per VM after the layout[^finding-2026-11-08] |
| Auth cipher        | `auth_service_cipher aes256k`: newer key format that old kernel clients cannot read |
| Quorum             | Survives one storage VM down; two down lose quorum           |

# Rules

- The MGR that is active can change. Use the [gateway](gateway.md) URL, not `https://<host>:8443/`.
- Keep the newer keys; do not relax authentication for older clients.

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

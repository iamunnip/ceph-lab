---
type: Ceph Cluster
title: Cluster
description: Ceph 20.2.5 tentacle on node-01 to node-03 with cephadm and Podman. Built by hand on 2026-11-08 and 2026-11-09, and with Ansible on 2026-11-09; the running cluster is the Ansible-built one. HEALTH_OK, 60 GiB raw.
tags: [ceph, cluster, cephadm]
resource: ../../../configs/podman/cluster-spec.example.yaml
status: stable
sources:
  - id: adr-0010
    resource: ../../../docs/podman/adr/0010-production-aligned-baseline.md
    title: "ADR 0010: Align the VM deployment with production practice"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
  - id: finding-2026-11-09
    resource: ../../../docs/podman/findings/2026-11-09-end-to-end.md
    title: "Finding 2026-11-09: End-to-end test, manual install then Ansible"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Method             | [podman](../methods/podman.md)                             |
| Ceph version       | 20.2.5 tentacle (stable), commit `ed27c686`              |
| Image              | `quay.io/ceph/ceph@sha256:4337ac8ac81d40df136e5eac92ca53bb52360a5e039f3d1f7935e729eb5ef42c` |
| Bootstrapped on    | [node-01](../nodes/node-01.md), `--mon-ip 192.168.56.11 --ssh-user vagrant --skip-monitoring-stack --skip-dashboard` |
| fsid               | Running cluster: `bcd7adcc-dc0d-11f1-b502-0800271f81c1` (Ansible, 2026-11-09, stronger-checks test). Earlier, destroyed: `73110d94-...` (Ansible, 2026-11-09), `8946ca06-...` (by hand, 2026-11-09), `81c36e54-...` (by hand, 2026-11-08). A new cluster gets a new one |
| Kernel             | Box kernel 7.0.0-34, no OS upgrade, since 2026-11-09[^finding-2026-11-09] |
| Public network     | `192.168.56.0/24`, no cluster network                    |
| Layout             | One spec file, `configs/podman/cluster-spec.example.yaml`, applied with `ceph orch apply -i` |
| Health             | `HEALTH_OK` about six to eight minutes after applying the layout[^finding-2026-11-08] |
| Raw space          | 60 GiB, 82 MiB used after the layout                       |
| Admin hosts        | node-01 and node-02 (label `_admin`)                     |
| Smoke test         | Pool `lab-smoke`: `rados put` and `get` read back the same file, SHA-256 `816728f4...8328` |
| Current state      | Running, `HEALTH_OK`, built with Ansible on 2026-11-09; the first dashboard login is not done yet |
| MDS and RGW        | Deferred. CephFS would add two MDS daemons on different storage hosts, subject to measured RAM |
| Production ready   | No. [ADR 0010](../../../docs/podman/adr/0010-production-aligned-baseline.md) lists the gaps[^adr-0010] |

# Rules

- Never bootstrap a cluster that exists. Bootstrap is skipped when `/etc/ceph/ceph.conf` exists.
- Recovery is never a wipe or a new bootstrap. To start over, recreate the VMs.
- Telemetry stays off.
- Short-lived `stray daemon(s) not managed by cephadm` warnings right after bootstrap cleared within about a minute.

# Data paths

- Native RBD on [node-04](../nodes/node-04.md): host nginx, ext4 at `/srv/cephdata`, the mapped image `native-rbd/native-data`, then the OSDs.
- k3s on [node-05](../nodes/node-05.md): a PVC makes the CSI provisioner create an RBD image; the CSI node plugin maps and mounts it; the pod writes through it to the OSDs.
- MONs hand out cluster maps and authentication. They never carry the data itself.

# Failure behavior

- One storage VM down: the other two MONs keep quorum, affected PGs degrade, I/O continues under `min_size 2`. The third copy cannot be rebuilt until the VM returns.
- Two storage VMs down: MON quorum is lost and normal service stops.

# Risks

- Every VM runs on one host computer.
- 6 GB RAM per storage VM once each has two OSDs.
- Small disks fill quickly.
- Client kernels may not read the newer `aes256k` keys.

# Learn more

- [Deploy the cluster by hand](../../../docs/podman/diataxis/how-to/deploy-vm-cluster.md)
- [Ceph HA and usable capacity](../../../docs/podman/diataxis/explanation/ha-and-capacity.md)

[^adr-0010]: ADR 0010: Align the VM deployment with production practice
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

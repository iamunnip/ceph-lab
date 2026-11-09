---
type: Tool
title: cephadm
description: The standalone Ceph tool that bootstraps the cluster, deploys every daemon as a container and opens cephadm shell. Used by the podman method.
tags: [tool, ceph, cephadm]
resource: https://docs.ceph.com/en/tentacle/cephadm/install/
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
| Version            | 20.2.5 tentacle (stable), commit `ed27c686770872022f4c1de04f6b0ac5d5384ded` |
| Download           | `https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm` (the same file for every Linux) |
| SHA-256            | `4cc603527d44ad8e753e1ed8a8e65cdcc05584fed74d1407425e0ae17ae13f55`[^adr-0004] |
| Installed at       | `/usr/local/sbin/cephadm`, `-rwxr-xr-x root root`, 1068332 bytes |
| Installed on       | node-01 and node-02 (group `ceph_admin`). The cluster copies its own copy to the other hosts |
| `cephadm check-host` | Podman 5.7.0, systemctl and lvcreate present, chrony running: `Host looks OK`[^finding-2026-11-08] |
| SSH                | Logs in to the other hosts as `vagrant` with the cluster key from `/etc/ceph/ceph.pub` |

# Rules

- Pinned versions are in `configs/podman/versions.yml`.
- Ubuntu 26.04 with Podman 5.7.0 is this lab's combination, not an upstream certification.

[^adr-0004]: ADR 0004: Pin and operate the VM deployment
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

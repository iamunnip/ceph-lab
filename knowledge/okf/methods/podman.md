---
type: Install Method
title: podman
description: Ceph 20.2.5 installed with the standalone cephadm, which runs every Ceph daemon as a Podman container from an image pinned by digest. Installed by hand on 2026-11-08.
tags: [method, cephadm, podman, install]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: stable
sources:
  - id: adr-0002
    resource: ../../../docs/podman/adr/0002-vm-topology.md
    title: "ADR 0002: VM cluster topology and cephadm"
  - id: adr-0013
    resource: ../../../docs/podman/adr/0013-podman-and-snap-tracks.md
    title: "ADR 0013: Separate podman and snap install methods"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field                   | Value                                                      |
|-------------------------|------------------------------------------------------------|
| Vagrantfile             | `vm/vagrant/podman/Vagrantfile`[^adr-0013]                 |
| VirtualBox name prefix  | `ceph-lab-podman-`                                         |
| Install tool            | [cephadm](../tools/cephadm.md) 20.2.5                      |
| Container runtime       | [Podman](../tools/podman.md) 5.7.0 from Ubuntu             |
| Pinned versions         | `configs/podman/versions.yml`                               |
| Layout files            | `configs/podman/cluster-spec.example.yaml`, `configs/podman/monitoring-spec.yaml` |
| Docs                    | `docs/podman/`                                             |
| Ansible                 | `ceph_install_method: podman`, settings in `ansible/vars/podman.yml` |
| Manual install          | Done on 2026-11-08, `HEALTH_OK`[^finding-2026-11-08]; repeated on new VMs on 2026-11-09 without an OS upgrade, `HEALTH_OK` |
| Ansible install         | Tested on new VMs on 2026-11-09: `HEALTH_OK`, second run `changed=0` |

# Why this method

- Ubuntu 26.04 ships only Ceph 20.2.0, and the upstream 20.2.5 packages do not cover Ubuntu 26.04. The MicroCeph snap had 20.2.1 on 2026-11-08. cephadm with Podman gives the pinned 20.2.5 on this OS[^finding-2026-11-08].

# Rules

- Uses the same VMs, names and addresses as [snap](snap.md). Only one of the two labs runs at a time.

# Learn more

- [Deploy the cluster by hand](../../../docs/podman/diataxis/how-to/deploy-vm-cluster.md)
- [Deploy with Ansible](../../../docs/podman/diataxis/how-to/deploy-with-ansible.md)

[^adr-0002]: ADR 0002: VM cluster topology and cephadm
[^adr-0013]: ADR 0013: Separate podman and snap install methods
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

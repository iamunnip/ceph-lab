---
type: Install Method
title: snap
description: Ceph installed from a snap package on the same VMs as the podman method. Planned. Only the Vagrantfile exists so far.
tags: [method, snap, install]
resource: ../../../vm/vagrant/snap/Vagrantfile
status: planned
sources:
  - id: adr-0013
    resource: ../../../docs/podman/adr/0013-podman-and-snap-tracks.md
    title: "ADR 0013: Separate podman and snap install methods"
---

# Facts

| Field                   | Value                                                      |
|-------------------------|------------------------------------------------------------|
| Vagrantfile             | `vm/vagrant/snap/Vagrantfile`, a copy of the podman one[^adr-0013] |
| VirtualBox name prefix  | `ceph-lab-snap-`                                           |
| VMs, addresses, disks   | Same as [podman](podman.md)                                |
| Snap, channel, revision | Not chosen yet. To be pinned in `configs/snap/versions.yml` |
| Docs                    | `docs/snap/`                                               |
| Ansible                 | `ceph_install_method: snap`. Stops at stage 0 until `snap_automation_ready: true` in `ansible/vars/snap.yml` |
| Manual install          | Not started                                                |

# Rules

- Only one of the two labs runs at a time: both use 192.168.56.11 to .15.
- Ansible follows only after the manual snap steps are verified.

# Learn more

- [Snap track](../../../docs/snap/README.md)

[^adr-0013]: ADR 0013: Separate podman and snap install methods

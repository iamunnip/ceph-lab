---
type: Ansible Role
title: setup
description: Shared host checks, then the method's own task file. Adds the lab hosts to /etc/hosts, checks the network and the disks without changing them, and for podman checks Podman and installs cephadm.
tags: [ansible, role, disk, network, cephadm]
resource: ../../../ansible/roles/setup
status: stable
sources:
  - id: adr-0012
    resource: ../../../docs/podman/adr/0012-ansible-roles.md
    title: "ADR 0012: Automate the VM install with three Ansible roles"
  - id: finding-2026-11-09
    resource: ../../../docs/podman/findings/2026-11-09-end-to-end.md
    title: "Finding 2026-11-09: End-to-end test, manual install then Ansible"
---

# Facts

| Field                | Value                                                         |
|----------------------|---------------------------------------------------------------|
| Folder               | `ansible/roles/setup/`                                      |
| Run by               | `ansible/site.yml` stage 2, on group `ceph`, tag `setup` |
| Method file          | `tasks/podman.yml` or `tasks/snap.yml` (snap stops with a message) |
| Tested               | 2026-11-09 on new VMs: all six disks passed as blank; on the finished cluster `/dev/sdb` passed as a Ceph OSD; second run `changed=0`[^finding-2026-11-09] |

# Steps

| File             | What it does                                                  |
|------------------|---------------------------------------------------------------|
| `hosts_file.yml` | Adds the [host name](../networks/host-names.md) lines, then checks `getent hosts` for the other storage VMs |
| `network.yml`  | Checks `ansible_host` is on `enp0s8`, pings the other storage VMs |
| `disks.yml`    | Checks the [data](../storage/data-disk.md) and [reserve](../storage/reserve-disk.md) disks exist, sit on their SCSI ports and are blank or Ceph OSDs |
| `podman.yml`   | Checks `podman --version` is 5.7.0; on `ceph_admin` hosts installs the pinned [cephadm](../tools/cephadm.md) with its SHA-256 and runs `cephadm check-host` |

# Safety

- Disk preparation is a check only. `wipefs --noheadings --output TYPE` (read only) must show nothing, or `LVM2_member` in a `ceph-...` VG. Anything else stops the run.

# Related

- Runs after [prereq](prereq.md), before [install_podman](install-podman.md).

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

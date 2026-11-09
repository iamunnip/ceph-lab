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
  - id: finding-stronger-checks
    resource: ../../../docs/podman/findings/2026-11-09-stronger-checks.md
    title: "Finding 2026-11-09: Stronger Ansible checks on new VMs"
---

# Facts

| Field                | Value                                                         |
|----------------------|---------------------------------------------------------------|
| Folder               | `ansible/roles/setup/`                                      |
| Run by               | `ansible/site.yml` stage 2, on group `ceph`, tag `setup`; `precheck.yml` again at the start of stage 3, tag `install` |
| Method file          | `tasks/podman.yml` or `tasks/snap.yml` (snap stops with a message) |
| Tested               | 2026-11-09 on new VMs: all six disks passed as blank; on the finished cluster `/dev/sdb` passed as a Ceph OSD; second run `changed=0`[^finding-2026-11-09] |

# Steps

| File             | What it does                                                  |
|------------------|---------------------------------------------------------------|
| `hosts_file.yml` | Adds the [host name](../networks/host-names.md) lines, then runs `hosts_check.yml` |
| `network.yml`  | Checks `ansible_host` is on `enp0s8`, pings the other storage VMs |
| `disks.yml`    | Checks the [data](../storage/data-disk.md) and [reserve](../storage/reserve-disk.md) disks exist, sit on their SCSI ports and are blank or Ceph OSDs |
| `hosts_check.yml` | Read only: `getent hosts` for the other storage VMs must give their host-only address |
| `podman_check.yml` | Read only: `podman --version` must be 5.7.0 |
| `cephadm_check.yml` | Read only: `cephadm version` must be the pinned version and commit; `cephadm check-host` must pass |
| `precheck.yml` | Read only: `hosts_check.yml`, `network.yml`, `disks.yml`, `podman_check.yml` and `cephadm_check.yml` again, before the install stage |
| `podman.yml`   | Runs `podman_check.yml`; on `ceph_admin` hosts installs the pinned [cephadm](../tools/cephadm.md) with its SHA-256, then runs `cephadm_check.yml` |

# Safety

- Disk preparation is a check only. `wipefs --noheadings --output TYPE` (read only) must show nothing, or `LVM2_member` in a `ceph-...` VG. For an LVM disk, `lvs --separator '|' -o lv_name,devices,lv_tags` must show exactly one LV on that disk only, whose tags name that LV in `ceph.block_device=` and hold `ceph.type=block`, `ceph.osd_id=<n>`, `ceph.osd_fsid=<uuid>` and exactly `ceph.cluster_fsid=` with the fsid from `/etc/ceph/ceph.conf` on the bootstrap host. Tags are compared as whole values. With no cluster yet, any LVM disk stops the run. Anything else stops the run.
- The per-LV tag check and `precheck.yml` were added on 2026-11-09 after the end-to-end test. On new VMs the same day they passed blank disks, then on the finished cluster passed `/dev/sdb` as an OSD of this cluster; second run and `--tags install` alone `changed=0`[^finding-stronger-checks]. The failing paths (mixed VG, longer fsid, LV spanning two disks, two LVs, tags naming another LV, no cluster) were tested on localhost only.

# Related

- Runs after [prereq](prereq.md), before [install_podman](install-podman.md).

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible
[^finding-stronger-checks]: Finding 2026-11-09: Stronger Ansible checks on new VMs

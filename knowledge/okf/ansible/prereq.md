---
type: Ansible Role
title: prereq
description: Shared by both methods. Refreshes the package lists and installs the packages of the chosen method. Never upgrades the OS and never reboots.
tags: [ansible, role, os, packages]
resource: ../../../ansible/roles/prereq
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
| Folder               | `ansible/roles/prereq/`                                     |
| Run by               | `ansible/site.yml` stage 1, on group `ceph`, tag `prereq` |
| Runs as              | root (`become: true`)                                       |
| Tested               | 2026-11-09 on new VMs; second run `changed=0`[^finding-2026-11-09] |
| Package refresh      | `cache_valid_time: 3600`, so a rerun within an hour skips `apt-get update` |
| Packages (podman)    | `podman lvm2 curl python3 chrony logrotate`, services `chrony` |

# Steps

| Step        | What it does                                                    |
|-------------|-----------------------------------------------------------------|
| Update      | `apt-get update`                                              |
| Packages    | `prereq_packages`, then `prereq_services` started and enabled |

# Rules

- No `apt-get upgrade`, no `full-upgrade` and no reboot. The VMs keep the box kernel 7.0.0-34. Dropped on 2026-11-09 at the user's request.

# Related

- Runs before [setup](setup.md).

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

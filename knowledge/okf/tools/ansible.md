---
type: Tool
title: Ansible
description: Runs the install from your computer over SSH. One playbook in ansible/ serves both methods, chosen with ceph_install_method.
tags: [tool, ansible, automation, host]
resource: ../../../ansible
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

| Field                  | Value                                               |
|------------------------|-----------------------------------------------------|
| Version                | ansible-core 2.22.0                                 |
| Runs on                | Your computer, from the `ansible/` folder (`ansible.cfg`) |
| Collections            | None. Only `ansible.builtin` modules[^adr-0012]     |
| Playbook               | `ansible/site.yml`: stage 0 check, then [prereq](../ansible/prereq.md), [setup](../ansible/setup.md), [install_podman](../ansible/install-podman.md) or [install_snap](../ansible/install-snap.md) |
| Method switch          | `ceph_install_method` in `inventory/group_vars/all.yml`, or `-e ceph_install_method=...` |
| Tested                 | 2026-11-09 on new VMs: `failed=0`, `HEALTH_OK`; second run `changed=0` on every VM; run on a hand-built cluster changed nothing in Ceph. Stage 0 stops on snap and on unknown values[^finding-2026-11-09] |
| SSH settings           | `timeout = 30` and `retries = 5` in `ansible.cfg`, because a run right after `vagrant up` timed out |

# Learn more

- [Deploy with Ansible](../../../docs/podman/diataxis/how-to/deploy-with-ansible.md)

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

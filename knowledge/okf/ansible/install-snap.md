---
type: Ansible Role
title: install_snap
description: The snap method's install. Not written yet; it stops with a message. Stage 0 already stops a snap run until snap_automation_ready is true.
tags: [ansible, role, snap, install]
resource: ../../../ansible/roles/install_snap
status: planned
sources:
  - id: adr-0012
    resource: ../../../docs/podman/adr/0012-ansible-roles.md
    title: "ADR 0012: Automate the VM install with three Ansible roles"
---

# Facts

| Field                | Value                                                         |
|----------------------|---------------------------------------------------------------|
| Folder               | `ansible/roles/install_snap/`                               |
| Settings             | `ansible/vars/snap.yml`, `snap_automation_ready: false`  |
| State                | Placeholder that fails with "not written yet"                 |

# Rules

- Written only from the verified manual snap steps in `docs/snap/`.

# Related

- The [snap](../methods/snap.md) method.

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles

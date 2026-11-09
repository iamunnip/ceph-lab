---
type: Ceph Service
title: Dashboard
description: The Ceph management UI, a module in the active MGR on port 8443. Opened through the gateway. Its certificate must come from the cephadm CA.
tags: [ceph, dashboard, ui, tls]
resource: ../../../docs/podman/diataxis/how-to/dashboard.md
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Module             | `dashboard`, enabled with `ceph mgr module enable dashboard` |
| Listens on         | The active MGR, `https://<host>:8443/`                   |
| Open with          | The [gateway](gateway.md): `https://192.168.56.11/`, `.12/` or `.13/` |
| Certificate        | From the cephadm CA: `ceph orch certmgr generate-certificates dashboard`, issuer `CN=cephadm-root-<fsid>` |
| First login        | User `admin`, role `administrator`, `--pwd_update_required` |
| First password     | `/root/ceph-dashboard-password` on node-01, mode 0600, deleted after the first login |

# Rules

- Never `ceph dashboard create-self-signed-cert`. The gateway rejects that certificate on port 29443, and the dashboard then shows **Error parsing Prometheus Alertmanager response: Expecting value**[^finding-2026-11-08].
- Never click **Add storage** on the welcome page. It appears even when OSDs exist, and it takes every spinning disk, including the [reserve disks](../storage/reserve-disk.md).
- Passwords, hashes and keys never go into git or chat.

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

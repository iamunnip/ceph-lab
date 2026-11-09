---
type: Ansible Role
title: install_podman
description: The podman method's install, run on node-01. Bootstraps once, copies the cluster SSH key, applies the layout, enables the dashboard, deploys monitoring and the shared gateway certificate, then waits for HEALTH_OK.
tags: [ansible, role, ceph, cephadm, install]
resource: ../../../ansible/roles/install_podman
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
| Folder               | `ansible/roles/install_podman/`                             |
| Run by               | `ansible/site.yml` stage 3, on group `ceph_bootstrap`, tag `install` |
| Other hosts          | `delegate_to` for the SSH key and the second admin host     |
| Commands             | `cephadm shell -- ceph ...`, as in the manual guide         |
| Templates            | `cluster-spec.yaml.j2`, `monitoring-spec.yaml.j2`; for the lab inventory they render the same as the files in `configs/podman/` |
| Settings             | `ansible/vars/podman.yml`                                   |
| Tested               | 2026-11-09 on new VMs: `failed=0`, `HEALTH_OK`; resumed after a failed run without bootstrapping again; second run `changed=0`[^finding-2026-11-09] |
| Monitoring wait      | `ceph orch ls <service> --refresh`, because without `--refresh` the list showed `0/N` for minutes after a manager restart |

# Steps

| File               | What it does                                                |
|--------------------|-------------------------------------------------------------|
| `bootstrap.yml`  | `cephadm bootstrap` unless `/etc/ceph/ceph.conf` exists; checks `public_network` |
| `ssh_access.yml` | Adds `/etc/ceph/ceph.pub` to `authorized_keys` on the other VMs, then `ceph cephadm check-host` |
| `cluster.yml`    | OSD memory settings, layout, waits for quorum and OSDs      |
| `admin_hosts.yml` | Waits for the admin keyring on node-02, `ceph health` from there |
| `dashboard.yml`  | Module, cephadm CA certificate, `admin` user with a random password in `/root/ceph-dashboard-password` |
| `monitoring.yml` | Image pins, monitoring layout, waits for every service      |
| `gateway_cert.yml` | Shared `CN=ceph-lab-gateway` certificate when not yet served |
| `verify.yml`     | Waits for `HEALTH_OK`, stops if a MON uses `10.0.2.15`  |

# Safety

- Each setting, certificate and user is checked first and changed only when needed.
- Passwords and private keys use `no_log: true` and stay on node-01.

# Checked on 2026-11-09

- `ceph mgr fail` without a name fails over the active manager.
- `ceph orch certmgr generate-certificates dashboard` prints plain JSON with `cert` and `key`.
- `openssl s_client` prints `subject=CN=ceph-lab-gateway`, which the gateway check matches.

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

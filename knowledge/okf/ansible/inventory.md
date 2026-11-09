---
type: Ansible Inventory
title: inventory
description: hosts.yml with the storage VMs, the client VMs and one group per Ceph label. Each host has ansible_host and vagrant_machine, which picks its Vagrant SSH key, also under delegate_to.
tags: [ansible, inventory, ssh]
resource: ../../../ansible/inventory
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

| Field                  | Value                                                          |
|------------------------|----------------------------------------------------------------|
| File                   | `ansible/inventory/hosts.yml`                                |
| Groups                 | `ceph` (node-01 to node-03), `ceph_bootstrap` (node-01), `ceph_admin` (node-01, node-02), `ceph_mon`, `ceph_mgr`, `ceph_osd`, `ceph_monitoring` (node-02), `ceph_gateway`, `clients` (node-04, node-05) |
| Ceph labels            | From the groups, mapped in `ceph_host_labels` (`_admin`, `mon`, `mgr`, `osd`, `monitoring`, `gateway`) |
| Host settings          | `ansible_host` (private network address), `vagrant_machine` (VM name in the Vagrantfile) |
| Method                 | `ceph_install_method: podman` in `group_vars/all.yml`      |
| SSH key                | `vm/vagrant/<method>/.vagrant/machines/<vagrant_machine>/virtualbox/private_key` |
| Why `vagrant_machine`  | With `delegate_to`, `inventory_hostname` keeps the original host's name, so node-01's key was offered to node-02: `Permission denied (publickey)` on 2026-11-09[^finding-2026-11-09] |
| Host keys              | Not checked or stored, like `vagrant ssh`                    |
| Versions               | `configs/<method>/versions.yml`, read as `lab_versions`     |
| Settings               | `group_vars/all.yml`, `group_vars/ceph.yml` (shared), `vars/podman.yml`, `vars/snap.yml`[^adr-0012] |

# Related

- The VMs: [node-01](../nodes/node-01.md) to [node-05](../nodes/node-05.md).

[^adr-0012]: ADR 0012: Automate the VM install with three Ansible roles
[^finding-2026-11-09]: Finding 2026-11-09: End-to-end test, manual install then Ansible

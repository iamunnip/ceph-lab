# Ansible for the three-VM Ceph lab

One playbook for both install methods. `ceph_install_method` in [inventory/group_vars/all.yml](inventory/group_vars/all.yml) chooses:

| Method | Installs Ceph with | Status |
| --- | --- | --- |
| `podman` (default) | cephadm, every daemon in a Podman container | Built from the [manual guide](../docs/podman/diataxis/how-to/deploy-vm-cluster.md); tested on new VMs on 2026-11-09, second run `changed=0` |
| `snap` | A Ceph snap | Not written yet. The playbook stops at stage 0 with a message |

See [Deploy with Ansible](../docs/podman/diataxis/how-to/deploy-with-ansible.md) for the step-by-step guide.

Run everything from this folder, so Ansible finds `ansible.cfg`:

```sh
ansible-playbook site.yml
```

To choose the method for one run only:

```sh
ansible-playbook site.yml -e ceph_install_method=podman
```

## Layout

```text
ansible.cfg                  # Ansible settings for this folder
site.yml                     # Stage 0 checks the method, then stages 1 to 3
inventory/
  hosts.yml                  # The VMs, addresses and groups (groups give Ceph labels)
  group_vars/
    all.yml                  # Install method, login, SSH key, network, pinned versions
    ceph.yml                 # Settings that are the same for every method
vars/
  podman.yml                 # Settings of the podman method
  snap.yml                   # Settings of the snap method (to fill in)
roles/
  prereq/                    # Stage 1, shared: apt-get update, install the method's packages
  setup/                     # Stage 2, shared: /etc/hosts, network and disk checks,
    tasks/podman.yml         #   then the method's file: Podman check and cephadm
    tasks/snap.yml           #   (snap: to write)
  install_podman/            # Stage 3, podman: bootstrap, layout, dashboard, monitoring
    templates/               #   cluster and monitoring layout files, built from the variables
  install_snap/              # Stage 3, snap: to write
```

The method also picks the Vagrant folder for the SSH keys (`vm/vagrant/<method>/`) and the pinned versions (`configs/<method>/versions.yml`).

Each task file starts with a comment that names the manual step it automates. Each task name ends with the command it runs, for example `(ceph orch apply -i)`.

## Rules the playbook follows

- It never wipes, partitions or formats a disk. It checks that each disk is blank or already a Ceph OSD, and stops otherwise.
- It never bootstraps twice, never upgrades the OS and never reboots a VM.
- Ceph uses only the host-only network 192.168.56.0/24, never the NAT address 10.0.2.15.
- Passwords and private keys stay on node-01 and are never printed (`no_log: true`).
- Versions come from `configs/<method>/versions.yml`, the same file the manual guide uses.
- Only `ansible.builtin` modules are used, so nothing else needs installing.

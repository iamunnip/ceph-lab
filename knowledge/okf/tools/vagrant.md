---
type: Tool
title: Vagrant
description: Creates, starts, stops and deletes the lab VMs from the Vagrantfile of the chosen method. Does not install any software inside them.
tags: [tool, vms, host]
resource: https://developer.hashicorp.com/vagrant
status: stable
sources:
  - id: adr-0009
    resource: ../../../docs/podman/adr/0009-vagrant-virtualbox-lab.md
    title: "ADR 0009: Run the VM lab on Vagrant and VirtualBox"
  - id: adr-0013
    resource: ../../../docs/podman/adr/0013-podman-and-snap-tracks.md
    title: "ADR 0013: Separate podman and snap install methods"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field                  | Value                                               |
|------------------------|-----------------------------------------------------|
| Version                | 2.4.9                                               |
| Runs on                | Your computer, in `vm/vagrant/podman/` or `vm/vagrant/snap/`[^adr-0013] |
| Reads                  | The `Vagrantfile` in the current folder             |
| Provider               | [VirtualBox](virtualbox.md)                         |
| Box                    | `cloud-image/ubuntu-26.04` 20260927.0.0             |
| Disks                  | Set with `vm.disk`, built in                        |
| State folder           | `.vagrant/` inside the method folder (VM IDs and SSH keys, not in git) |
| Shared folder          | `/vagrant` disabled                                 |
| Provisioners           | None. Ceph is installed by hand or with [Ansible](ansible.md) |

# Rules

- Removing a disk line from the Vagrantfile deletes that disk on the next `vagrant reload`[^finding-2026-11-08]. Never do it while the VM exists.
- `vagrant destroy` deletes the VMs and every disk. It is the only way to start over; disks are never wiped by hand.

# Open questions

- `vagrant reload` keeps disks, addresses, host names and Ceph daemons. `vagrant halt` and `vagrant up` are not tested yet.

# Learn more

- [Create the VMs](../../../docs/podman/diataxis/how-to/create-vagrant-vms.md)

[^adr-0013]: ADR 0013: Separate podman and snap install methods
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

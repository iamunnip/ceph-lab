---
type: Network
title: Private network
description: The 192.168.56.0/24 VirtualBox host-only network shared by all VMs and your computer. Ceph's only network, also used by Ansible and the dashboard.
tags: [network, virtualbox, host-only, ceph]
resource: ../../../vm/vagrant/podman/Vagrantfile
status: stable
sources:
  - id: adr-0009
    resource: ../../../docs/podman/adr/0009-vagrant-virtualbox-lab.md
    title: "ADR 0009: Run the VM lab on Vagrant and VirtualBox"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field            | Value                                                  |
|------------------|--------------------------------------------------------|
| Range            | `192.168.56.0/24`                                      |
| VirtualBox type  | Host-only, `vboxnet0`                                  |
| Card in each VM  | Card 2, `enp0s8`, VirtIO (`nic_type: "virtio"`), MTU 1500 |
| Your computer    | `192.168.56.1`                                         |
| Used addresses   | `.11` to `.13` storage VMs, `.14` [node-04](../nodes/node-04.md), `.15` [node-05](../nodes/node-05.md) |
| Shared by        | Both [methods](../methods/index.md), so only one lab runs at a time |
| Ceph public network | `192.168.56.0/24`, derived from `--mon-ip 192.168.56.11`; no separate cluster network[^finding-2026-11-08] |
| Ceph ports       | MON TCP 3300 and 6789; OSD and other daemons TCP 6800 to 7568; [gateway](../ceph/gateway.md) TCP 443 |
| Set by           | The `private_network` line in `vm/vagrant/<method>/Vagrantfile` |

# Used for

- All Ceph traffic between the storage VMs.
- Your computer to the VMs: Ansible on port 22, the dashboard on port 443.

# Rules

- Ceph must use this network, never the [NAT network](nat.md).
- Not reachable from other computers on your home network. Do not forward port 443 or any Ceph port.

# Open questions

- Partial: all three MONs bind to 192.168.56.x and the gateway listens on 0.0.0.0:443. The OSD listeners are still to check.

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

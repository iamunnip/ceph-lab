---
type: Network
title: NAT network
description: The 10.0.2.0/24 VirtualBox NAT network inside each VM. Used for internet access and vagrant ssh, never for Ceph.
tags: [network, virtualbox, nat]
status: stable
---

# Facts

| Field            | Value                                                    |
|------------------|----------------------------------------------------------|
| Range            | `10.0.2.0/24`                                            |
| VM address       | `10.0.2.15` on **every** VM                              |
| Card in each VM  | Card 1, `enp0s3`                                         |
| Default route    | Yes. Internet traffic leaves through this card           |
| Set by           | VirtualBox and Vagrant, automatically                    |

# Used for

- Downloading packages, cephadm and container images.
- `vagrant ssh`, through a forwarded port on `127.0.0.1`.

# Rules

- Ceph must not use it: every VM has the same address. Bootstrap passes `--mon-ip 192.168.56.11` so Ceph picks the [private network](private-network.md).
- The Ansible [install_podman](../ansible/install-podman.md) role stops if a monitor uses `10.0.2.15`.

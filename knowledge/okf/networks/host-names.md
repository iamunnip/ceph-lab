---
type: Network
title: Host names
description: Every storage VM must resolve every other lab VM by name. Vagrant only maps a VM's own name, so the lab adds five lines to /etc/hosts.
tags: [network, dns, hosts]
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| From Vagrant       | Own name only, mapped to `127.0.1.1`                       |
| Lines added        | `# ceph-lab hosts`, then `192.168.56.11 node-01` to `192.168.56.15 node-05` |
| Added by           | The manual guide, or the Ansible [setup](../ansible/setup.md) role (`lineinfile`, from the inventory) |
| Survives           | `vagrant reload` (checked on node-03 on 2026-11-08)[^finding-2026-11-08] |

# Why it is needed

- cephadm writes the [gateway](../ceph/gateway.md) upstreams as `<hostname>:8443`. Without these lines nginx fails with `host not found in upstream "node-01:8443"` and the cluster shows `CEPHADM_FAILED_DAEMON`[^finding-2026-11-08].
- MONs, MGRs and OSDs were not affected, because cephadm addresses them by IP.

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

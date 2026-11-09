---
type: Tool
title: VirtualBox
description: The hypervisor on your computer that runs the lab VMs, their disks and the host-only network.
tags: [tool, vms, host]
resource: https://www.virtualbox.org/
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

| Field                  | Value                                               |
|------------------------|-----------------------------------------------------|
| Version                | 7.2.22r175245                                       |
| Host computer          | 12 CPU threads, about 40 GB RAM (38 GiB seen), about 113 GB free disk[^finding-2026-11-08] |
| Lab size               | 5 VMs: 10 CPUs, 30 GB memory. One method's lab at a time |
| VM names               | `ceph-lab-podman-<node>` or `ceph-lab-snap-<node>`  |
| Host-only network      | `vboxnet0`, `192.168.56.1`                          |
| IO-APIC                | Turned on by the Vagrantfile; the box ships with it off, which limits a VM to 1 CPU |
| Guest Additions        | Box has 6.0.0, a mismatch warning only. Shared folders are disabled, so it has no effect |

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

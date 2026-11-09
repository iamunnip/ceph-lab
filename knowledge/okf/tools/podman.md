---
type: Tool
title: Podman
description: The container runtime of the podman method. Every Ceph daemon runs as a Podman container under its own systemd unit. No background daemon.
tags: [tool, containers, podman]
resource: https://podman.io/
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Version            | 5.7.0 (`5.7.0+ds2-3build1` from Ubuntu 26.04)[^finding-2026-11-08] |
| Installed with     | `podman` and 30 helper packages such as `crun`, `conmon` and `netavark` |
| Ceph image         | `quay.io/ceph/ceph@sha256:4337ac8a...ef42c`, 1.52 GB       |
| Started by         | systemd units `ceph-<fsid>@<daemon>.service`, at boot and after a failure |
| Checked by         | The Ansible [setup](../ansible/setup.md) role stops if `podman --version` is not 5.7.0 |

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

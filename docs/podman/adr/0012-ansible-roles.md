# 0012: Automate the VM install with three Ansible roles

Status: accepted; tested on new VMs on 2026-11-09, see the [finding](../findings/2026-11-09-end-to-end.md). Date: 2026-11-09.

## Context

The manual install in the [deployment guide](../diataxis/how-to/deploy-vm-cluster.md) and the [dashboard guide](../diataxis/how-to/dashboard.md) is recorded command by command. The next stage turns it into repeatable automation (an open question, now on the [install_podman](../../../knowledge/okf/ansible/install-podman.md) knowledge page). The automation must stay readable for beginners, keep every value in variables, and follow the lab's safety rules: never wipe a disk, never bootstrap twice, Ceph only on 192.168.56.0/24.

## Decision

- One playbook at the repository root, [ansible/site.yml](../../../ansible/site.yml), run from the host computer, for every install method. `ceph_install_method` (`podman` or `snap`) chooses the method ([ADR 0013](0013-podman-and-snap-tracks.md)). Stage 0 checks the method and loads `vars/<method>.yml`. Then three stages, one role each:
  - `prereq` (shared): `apt-get update` and the packages the method needs. No OS upgrade and no reboot.
  - `setup` (shared): `/etc/hosts`, network checks and read-only disk checks, then the method's own task file. For podman that is the Podman version check and the pinned cephadm on the admin hosts.
  - `install_<method>`: for podman, everything from bootstrap to the shared gateway certificate, run on the bootstrap host, with `delegate_to` for the SSH key and the second admin host. `install_snap` stops with a message until the snap steps are written.
- Ceph labels come from inventory groups (`ceph_mon`, `ceph_osd`, `ceph_gateway` and so on). The cluster and monitoring layout files are templates built from the inventory and group variables.
- Settings shared by every method live in `inventory/group_vars/all.yml` and `ceph.yml`; settings of one method live in `vars/<method>.yml`. Roles have no defaults of their own, so a beginner has only these files to look at.
- Pinned versions are read from `configs/<method>/versions.yml`, for podman [configs/podman/versions.yml](../../../configs/podman/versions.yml), instead of being copied.
- Only `ansible.builtin` modules and no `shell` module. Each task runs one command from the manual guide, and its name says which.
- Disk preparation is a check only. A disk must be blank or already a Ceph OSD; anything else stops the run.
- Ansible logs in with the per-VM keys Vagrant creates and, like `vagrant ssh`, does not store host keys.

## Consequences

- The manual guide stays the reference. A change to a manual step needs the same change in the matching task.
- The layout templates repeat what `cluster-spec.example.yaml` and `monitoring-spec.yaml` hold. For the lab inventory they render the same content.
- `ceph orch apply` runs on every run. That is safe because applying the same layout changes nothing in Ceph.
- Rebooting a VM that already runs Ceph is left to the operator, one VM at a time.
- The clients on node-04 and node-05 are not automated yet.
- A second method adds a `vars/<method>.yml`, a `roles/setup/tasks/<method>.yml` and an `install_<method>` role; the shared roles stay as they are.

## Relation to earlier decisions

Follows [ADR 0009](0009-vagrant-virtualbox-lab.md), which planned Ansible after the manual install, and keeps the direct Ceph commands of [ADR 0006](0006-direct-cli-validation.md).

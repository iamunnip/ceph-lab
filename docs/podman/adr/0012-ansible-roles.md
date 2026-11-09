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
- Disk preparation is a check only. A disk must be blank or already an OSD of this cluster: it must hold exactly one logical volume, on that disk only, whose tags name that volume as `ceph.block_device` and hold `ceph.type=block`, an OSD id, an OSD fsid and exactly the fsid from `/etc/ceph/ceph.conf` on the bootstrap host. Tags are compared as whole values. Anything else stops the run. (Added 2026-11-09; before, a `ceph-...` volume group name was enough.)
- The install stage first runs the setup role's read-only checks again (`roles/setup/tasks/precheck.yml`: host names, network, disks, Podman version, cephadm and `cephadm check-host`) on every storage VM, so a staged `--tags install` run checks its current targets before bootstrap and OSD creation. (Added 2026-11-09.)
- The gateway certificate is checked on every gateway, not by name alone: common name, at least `ceph_gateway_cert_min_days` (30) days left, every gateway name and address as a whole entry, and one fingerprint on all gateways. A failed check makes a new certificate. (Added 2026-11-09.)
- The final check compares the layout with the inventory by name and place: the quorum names, the hosts with a running manager, every OSD up and in, and `host:disk` of every OSD against `ceph_osd_devices`. The reserve disks count only when `ceph_reserve_devices_in_ceph` is true. (Added 2026-11-09.)
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

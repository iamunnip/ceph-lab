# 0013: Separate podman and snap install methods

Status: accepted. Date: 2026-11-09.

## Context

The user wants a second way to install Ceph on the VM lab, from a snap, next to the existing cephadm install that runs every daemon in Podman. Each method needs its own Vagrantfile, and the two must be easy to tell apart. Until now every VM file sat under `vm/`.

The JSON files were also reviewed. `vm/inventory/vagrant.json` was read by nothing. Its planned values repeated the Vagrantfile, it still listed the old 30 GB OS disk, and its observed values are in the [2026-11-08 finding](../findings/2026-11-08-vagrant-vms.md). `vm/config/versions.json` is read by Ansible, but JSON cannot hold comments.

## Decision

The user chose this layout:

- `vm/vagrant/podman/Vagrantfile` and `vm/vagrant/snap/Vagrantfile`, and nothing else under `vm/`. Both use the same VM names, addresses and disks, so the guides keep the lab's real values. VirtualBox names get a method prefix, `ceph-lab-podman-` or `ceph-lab-snap-`. Only one lab runs at a time.
- One `ansible/` at the repository root for both methods, chosen with `ceph_install_method` ([ADR 0012](0012-ansible-roles.md)).
- `configs/<method>/` at the root for what the manual guides and Ansible both read: for podman `versions.yml`, the cephadm layout files and the client files.
- `docs/<method>/` at the root for the guides and decisions of each method. `docs/README.md` is the shared documentation guide.
- `knowledge/okf/` at the root, shared by both methods: a facts-only OKF bundle in the layout of the user's kube-lab repository, with `index.md`, `log.md` and one folder per topic. The full experiment records move to `docs/<method>/findings/`.
- `vm/inventory/vagrant.json` is removed. Observed machine facts stay in the findings.
- `versions.json` becomes `configs/podman/versions.yml`, with the same values and comments.

## Consequences

- Switching methods means `vagrant destroy` in one Vagrant folder and `vagrant up` in the other, then setting `ceph_install_method` to match. That deletes the first lab's Ceph data.
- Commands in the guides that upload files from the Vagrant folder use `../../../configs/podman/...`. Findings keep the commands exactly as they were run on 2026-11-08, with the old paths.
- Decisions for both methods, like this one, are recorded with the podman ADRs, where the history is.
- VM creation and preflight are the same for both methods. For now the snap docs link to the podman guides for them.

## Relation to earlier decisions

Partly supersedes [ADR 0004](0004-vm-deployment-baseline.md) (version and inventory files), [ADR 0007](0007-vm-owned-docs-and-dashboard.md) (documentation location) and [ADR 0009](0009-vagrant-virtualbox-lab.md) (Vagrantfile path).

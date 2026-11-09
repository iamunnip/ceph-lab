# Snap track

This track installs Ceph on the lab VMs from a snap package, instead of with cephadm and Podman as in the [podman track](../podman/README.md). It lets you compare the two methods on the same VMs and disks.

Status: planned. Only the [Vagrantfile](../../vm/vagrant/snap/Vagrantfile) exists so far. Write the install steps by hand first, one command at a time, as in the podman track. Ansible follows once they are verified.

## Same VMs as the podman track

The Vagrantfile is a copy of the podman one, with the same VM names, addresses and disks:

| Host | Address | Extra disks |
| --- | --- | --- |
| node-01 | 192.168.56.11 | 20 GB data `/dev/sdb`, 20 GB reserve `/dev/sdc` |
| node-02 | 192.168.56.12 | same |
| node-03 | 192.168.56.13 | same |
| node-04 | 192.168.56.14 | none (optional client) |
| node-05 | 192.168.56.15 | none (optional client) |

Only the VirtualBox names differ: `ceph-lab-snap-node-01` here, `ceph-lab-podman-node-01` in the podman track.

**Run only one lab at a time.** Both use the same addresses on the same host-only network, and together they need more memory than the host computer has. Before `vagrant up` in `vm/vagrant/snap/`, run `vagrant destroy` in `vm/vagrant/podman/`, and the other way round.

The VM steps are the same as in the podman track: [create the VMs](../podman/diataxis/how-to/create-vagrant-vms.md) and run the [preflight](../podman/diataxis/how-to/vm-preflight.md). Run them from `vm/vagrant/snap/` instead of `vm/vagrant/podman/`.

## What to add

| Where | What |
| --- | --- |
| `docs/snap/diataxis/how-to/` | The manual install, dashboard and reserve disk guides |
| `docs/snap/adr/` | Decisions, starting with which snap, channel and revision to use |
| `configs/snap/versions.yml` | The pinned snap, channel and revision, in the same format as [configs/podman/versions.yml](../../configs/podman/versions.yml) |
| [ansible/vars/snap.yml](../../ansible/vars/snap.yml) | The Ansible settings, then set `snap_automation_ready: true` |
| `ansible/roles/setup/tasks/snap.yml` and `ansible/roles/install_snap/` | The automation, built from the verified manual steps |

Record the compatibility evidence and every experiment in the shared [knowledge bundle](../../knowledge/okf/index.md).

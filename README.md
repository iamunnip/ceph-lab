# ceph-lab

A hands-on lab for learning Ceph on virtual machines and Kubernetes, including Rook.

The VM lab runs on Vagrant and VirtualBox: three storage VMs with a 20 GB data disk and a 20 GB reserve disk each, plus two optional client VMs. Ceph can be installed on it in two ways:

| Method | How Ceph is installed | Vagrantfile | Docs | Status |
| --- | --- | --- | --- | --- |
| podman | cephadm, which runs every Ceph daemon as a Podman container from a pinned image | [vm/vagrant/podman/](vm/vagrant/podman/Vagrantfile) | [docs/podman/](docs/podman/README.md) | Manual install and Ansible both tested end to end on 2026-11-09 |
| snap | A Ceph snap package | [vm/vagrant/snap/](vm/vagrant/snap/Vagrantfile) | [docs/snap/](docs/snap/README.md) | Planned: only the Vagrantfile so far |

One [Ansible playbook](ansible/README.md) serves both methods. Set `ceph_install_method` to `podman` or `snap`. Kubernetes and Rook deployment tracks are planned for later.

## Repository structure

```text
vm/vagrant/
  podman/Vagrantfile   # the lab VMs for the podman method
  snap/Vagrantfile     # the same VMs for the snap method
ansible/               # one playbook for both methods
  inventory/           # the VMs, groups and shared settings
  vars/                # podman.yml and snap.yml: settings of each method
  roles/               # prereq and setup (shared), install_podman, install_snap
configs/
  podman/              # pinned versions (versions.yml), cephadm layout files, client files
docs/                  # documentation, one folder per method
  podman/              # diataxis guides, ADRs and findings
  snap/                # planned
knowledge/okf/         # shared questions, facts and experiment findings
k8s/                   # future Kubernetes track
rook/                  # future Rook track
```

## Run one lab at a time

Both Vagrantfiles use the same VM names (node-01 to node-05) and the same addresses (192.168.56.11 to .15), so the guides work for either method without changes. VirtualBox tells them apart by name: `ceph-lab-podman-node-01` or `ceph-lab-snap-node-01`.

Two labs cannot run at once: the addresses would clash, and six storage VMs need 36 GB RAM. To switch, destroy the running lab from its own Vagrant folder first, for example:

```sh
cd vm/vagrant/podman
```

```sh
vagrant destroy -f
```

This deletes those VMs and all their disks, including the Ceph data. Then run `vagrant up` in the other method's folder, and set `ceph_install_method` to match before running Ansible.

## Start here

Pick a method, create its VMs with `vagrant up` from its folder under `vm/vagrant/`, then follow its guides, starting with the [podman track](docs/podman/README.md). The guides use the lab's real addresses and disk paths, so commands can be copied as they are. See the [knowledge bundle](knowledge/okf/index.md) and the [documentation guide](docs/README.md) for how changes and findings are recorded.

## License

Original code, configurations, and documentation are licensed under the [MIT License](LICENSE).
Third-party material retains its original license and notices.

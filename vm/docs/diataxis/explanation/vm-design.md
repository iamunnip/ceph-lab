# VM design rationale

The lab runs on Vagrant and VirtualBox, so anyone can rebuild the same five VMs from one [Vagrantfile](../../../vagrant/Vagrantfile) on their own computer. `vagrant destroy` and `vagrant up` give a clean start without wiping disks by hand. See [ADR 0009](../../adr/0009-vagrant-virtualbox-lab.md).

Ceph is installed by hand first, so each step can be understood and checked. The verified manual steps then become Ansible automation.

Cephadm keeps the first learning track focused on Ceph rather than Kubernetes scheduling. Three storage hosts let the lab explore monitor quorum and host-level replica placement. One OSD per host makes failures easy to observe, but gives no spare host for restoring a third replica during a host outage.

Each storage VM has a 20 GiB data disk and a 20 GiB reserve disk. The small first OSD lets the cluster reach its nearfull warning with little data. The reserve disk is attached from the start but left out of Ceph, so the low-space exercise needs no change to the Vagrantfile. Vagrant deletes disks that are removed from the Vagrantfile, so disks are never added and removed there to simulate growth.

Each VM has 2 CPUs and 6 GB RAM, so all five fit on a host with 12 CPU threads and 40 GB RAM. OSD memory targets are lowered to match.

Ceph uses the host-only network on `enp0s8`. The NAT card has the same address on every VM and is only for internet access and `vagrant ssh`. Client access requires reachable OSDs as well as monitors; reaching a MON alone is insufficient.

RBD comes first because it exposes image lifecycle, client authentication and block-device persistence. CephFS adds metadata services and shared filesystem behavior. The native client on node-04 and the single-node k3s client on node-05 consume the same external Ceph cluster, showing that CSI integrates Kubernetes with storage independently of where Ceph runs.

Start with replicated pools, not erasure coding, to keep the first experiments understandable and compatible with three hosts. Use scoped CephX identities for native services and CSI; admin credentials are for administration only. One host computer running every VM does not establish production readiness or HA for data or applications.

See the [accepted decisions](../../adr/README.md) and [VM reference](../reference/vm-lab.md).

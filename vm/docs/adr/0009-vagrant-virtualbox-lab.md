# 0009: Run the VM lab on Vagrant and VirtualBox

Status: accepted; VMs created and checked. Date: 2026-11-08.

## Context

The user wants a lab that runs on their own computer, with 12 CPU threads and about 40 GB RAM, and that anyone can rebuild the same way. The first Ceph install is done by hand to learn each step; Ansible automation comes after. The lab must also show what happens when Ceph runs low on space.

## Decision

Define the VMs in [vm/vagrant/Vagrantfile](../../vagrant/Vagrantfile), following the layout of the user's kube-lab Vagrantfile:

- Box `cloud-image/ubuntu-26.04` on VirtualBox, with the shared `/vagrant` folder disabled.
- Five VMs, node-01 to node-05, each with 2 CPUs and 6 GB RAM, 10 CPUs and 30 GB in total. `vagrant up` starts only the three storage VMs; the clients start by name.
- The OS disk is grown from the box's 10 GB to 20 GB. 10 GB would leave about 7.5 GB free, too little for the Ceph and monitoring images, logs and up to 2 GB of Prometheus data. 20 GB leaves about 17 GB free.
- Each storage VM has a 20 GB data disk on SCSI port 1 (`/dev/sdb`) and a 20 GB reserve disk on port 2 (`/dev/sdc`). The reserve disk is attached from the start and left out of Ceph until the low-space exercise.
- A host-only network with fixed addresses 192.168.56.11 to 15 on a virtio card carries all Ceph traffic. The NAT card is only for internet access and `vagrant ssh`.
- The IO-APIC is turned on so the VMs can use more than one CPU.
- Vagrant only creates VMs. Ceph is installed by hand from the [deployment guide](../diataxis/how-to/deploy-vm-cluster.md), then automated with Ansible in a later stage.

## Why the reserve disk is attached from the start

Vagrant deletes a disk that is removed from the Vagrantfile on the next reload. Adding and removing disks through the Vagrantfile could therefore destroy an OSD. Attaching both disks once and choosing in Ceph which to use avoids that.

## Consequences

- The lab runs anywhere VirtualBox and Vagrant run. `vagrant destroy` and `vagrant up` give a clean start, so no disk is ever wiped by hand.
- All VMs share one host computer: Ceph host failure can be simulated, but a host computer failure loses everything.
- Disk sizes are small on purpose: about 20 GiB usable at first and 40 GiB after adding the reserve disks.
- VirtualBox disks have no serial number; disks are identified by SCSI port.
- 6 GB per storage VM is tight for two OSDs plus MON, MGR and monitoring; memory must be measured after the reserve disks are added.

## Evidence

[2026-11-08 VM finding](../../knowledge/okf/findings/2026-11-08-vagrant-vms.md).

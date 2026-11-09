---
okf_version: "0.2"
---

# About this bundle

* [Log](log.md) - History of changes to this bundle, newest first.

This bundle holds facts only: what the lab is, how it is built and what was observed. Guides, decisions and the full experiment records are in [docs/](../../docs/README.md).

# Folders

* [methods](methods/index.md) - The two ways to install Ceph: podman (tested by hand and with Ansible) and snap (planned).
* [nodes](nodes/index.md) - One page per VM: node-01 to node-05.
* [storage](storage/index.md) - The OS disk, the data disk and the reserve disk.
* [networks](networks/index.md) - The private network, the NAT network and host name resolution.
* [ceph](ceph/index.md) - The cluster, its daemons, pools, dashboard, gateway and monitoring.
* [ansible](ansible/index.md) - The inventory and the roles: prereq, setup, install_podman, install_snap.
* [tools](tools/index.md) - VirtualBox, Vagrant, Ansible, cephadm and Podman: what each does in the lab.

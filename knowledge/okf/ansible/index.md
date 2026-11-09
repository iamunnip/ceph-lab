# Ansible Role

* [inventory](inventory.md) - hosts.yml: the VMs, groups that give Ceph labels, and the method switch.
* [prereq](prereq.md) - Shared: apt-get update and the method's packages. No OS upgrade, no reboot.
* [setup](setup.md) - Shared: /etc/hosts, network and disk checks, then the method's tools (podman: Podman check and cephadm).
* [install_podman](install-podman.md) - Bootstrap, SSH key, layout, dashboard, monitoring, gateway certificate, HEALTH_OK.
* [install_snap](install-snap.md) - Not written yet. Stops with a message.

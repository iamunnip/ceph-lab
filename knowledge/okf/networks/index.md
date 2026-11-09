# Network

* [Private network](private-network.md) - 192.168.56.0/24 host-only network. The only network Ceph uses.
* [NAT network](nat.md) - 10.0.2.0/24 inside each VM. Internet and `vagrant ssh` only, never Ceph.
* [Host names](host-names.md) - The `/etc/hosts` lines every storage VM needs to resolve the others.

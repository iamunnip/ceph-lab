# VM lab reference

The lab is five VirtualBox VMs created by the [Vagrantfile](../../../../vm/vagrant/podman/Vagrantfile). The [observed inventory](../../findings/2026-11-08-vagrant-vms.md) records what the VMs reported on 2026-11-08.

| VM | Address | Role | Extra disks | Planned daemons and software |
| --- | --- | --- | --- | --- |
| node-01 | 192.168.56.11 | Storage, first administration host | 20 GiB data `/dev/sdb`, 20 GiB reserve `/dev/sdc` | MON, MGR, OSD |
| node-02 | 192.168.56.12 | Storage, second administration host | same | MON, MGR, OSD, Prometheus, Alertmanager, Grafana, gateway |
| node-03 | 192.168.56.13 | Storage | same | MON, MGR, OSD |
| node-04 | 192.168.56.14 | Normal VM using Ceph | none | 5 GiB RBD image, ext4 at /srv/cephdata, host nginx |
| node-05 | 192.168.56.15 | Single-node k3s using Ceph | none | k3s, Ceph-CSI, 5 GiB RWO PVC, nginx |

Every VM: 2 vCPUs, 6 GB RAM, 20 GB OS disk `/dev/sda`, Ubuntu 26.04.1 LTS, kernel 7.0.0-34-generic from the box (not upgraded), SSH user `vagrant` with passwordless sudo. `vagrant up` starts node-01 to node-03; node-04 and node-05 are started by name.

CephFS adds two MDS daemons on different storage hosts (one active, one standby), subject to measured RAM. RGW is optional and has no HA commitment.

## Storage and failure policy

- One whole data device per storage VM, `/dev/sdb`, selected by path in the [cluster spec](../../../../configs/podman/cluster-spec.example.yaml). The OS disk `/dev/sda` is never used.
- The reserve disk `/dev/sdc` is added only by the [reserve disk exercise](../how-to/add-reserve-disks.md). Never apply `--all-available-devices`.
- Disks are identified by VirtIO SCSI port: port 1 (`scsi-0:0:1:0`) is the data disk, port 2 (`scsi-0:0:2:0`) the reserve disk. VirtualBox gives these disks no serial number, and the OS disk is also 20 GiB, so size never identifies a disk.
- Replicated pools: `size=3`, `min_size=2`, CRUSH failure domain `host`.
- Raw capacity 60 GiB at first, 120 GiB with the reserve disks. Usable with three copies: about 20 GiB, then about 40 GiB.
- One storage-host outage should keep two-monitor quorum and allow eligible I/O while degraded. With only three hosts, the third copy cannot be rebuilt until the host returns.
- Two storage-host outages lose MON quorum and stop normal service.

## Network contract

| Network | Interface | Addresses | Use |
| --- | --- | --- | --- |
| VirtualBox NAT | `enp0s3` | 10.0.2.15 on every VM | Internet and `vagrant ssh` only |
| VirtualBox host-only `vboxnet0` | `enp0s8` | 192.168.56.11 to 15, host 192.168.56.1 | Ceph public network, SSH between VMs, dashboard |

Ceph has one public network, 192.168.56.0/24, and no separate cluster network. Monitor ports are TCP 3300 and 6789, OSD and other daemons use TCP 6800 to 7568, and the dashboard gateway uses TCP 443 on node-01, node-02 and node-03. The host-only network is reachable only from the host computer.

## Version contract

Ceph and cephadm 20.2.5, Podman 5.7.0 and digest-pinned monitoring images; see [version pins](../../../../configs/podman/versions.yml). Client versions: k3s v1.36.6+k3s1, Ceph-CSI 3.18.2, Helm v3.19.0. Host tools: Vagrant 2.4.9, VirtualBox 7.2.22, box `cloud-image/ubuntu-26.04` 20260927.0.0. Check supported combinations in the selected release documentation. Upstream `latest` documentation is orientation only and may describe development releases.

## Dashboard

The UI is part of the core setup. It runs in the active MGR, behind a cephadm `mgmt-gateway` on HTTPS port 443 of every storage VM, with one self-signed certificate (`CN=ceph-lab-gateway`) that names all three hosts and addresses. Open `https://192.168.56.11/`, `.12/` or `.13/` from the host computer. Follow the [dashboard guide](../how-to/dashboard.md). Do not expose MON or OSD ports.

## Sources

- [Cephadm deployment requirements](https://docs.ceph.com/en/tentacle/cephadm/install/)
- [Vagrant disks with VirtualBox](https://developer.hashicorp.com/vagrant/docs/disks/virtualbox/usage)
- [Ceph-CSI project and compatibility information](https://github.com/ceph/ceph-csi)
- [Ceph RBD with CSI](https://docs.ceph.com/en/tentacle/rbd/rbd-kubernetes/)

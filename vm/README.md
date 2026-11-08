# Ceph on three VMs with Vagrant

Set up a Ceph cluster on three VirtualBox VMs created by Vagrant, use each VM's data disk for storage, and enable the Ceph Dashboard. Two more VMs use the cluster's storage. Ceph is installed by hand first; Ansible automation comes later.

## Start here

1. [Create the VMs with Vagrant](docs/diataxis/how-to/create-vagrant-vms.md).
2. [Check the VMs](docs/diataxis/how-to/vm-preflight.md).
3. [Deploy Ceph by hand](docs/diataxis/how-to/deploy-vm-cluster.md).
4. [Enable and open the dashboard](docs/diataxis/how-to/dashboard.md) at `https://192.168.56.11/`, `.12/` or `.13/`.
5. [Fill the cluster and add the reserve disks](docs/diataxis/how-to/add-reserve-disks.md).

## Machines

Every VM has 2 CPUs, 6 GB RAM and a 20 GB OS disk. All five together need 10 CPUs and 30 GB RAM.

| Host | Address | Purpose | Extra disks |
| --- | --- | --- | --- |
| node-01 | 192.168.56.11 | MON, MGR, OSD and administration | 20 GB data `/dev/sdb`, 20 GB reserve `/dev/sdc` |
| node-02 | 192.168.56.12 | MON, MGR, OSD, second administration host, monitoring and HTTPS gateway | same |
| node-03 | 192.168.56.13 | MON, MGR and OSD | same |
| node-04 | 192.168.56.14 | Normal VM using a Ceph block device (optional) | none |
| node-05 | 192.168.56.15 | Single-node k3s using Ceph through CSI (optional) | none |

`vagrant up` starts node-01 to node-03. Start the clients by name when you need them.

Both extra disks are for Ceph data. The data disk is the first OSD. The reserve disk stays empty until the cluster runs low on space, then becomes a second OSD on each host. The cluster uses three replicas across hosts, with a minimum of two for I/O: 60 GiB raw gives about 20 GiB usable at first, and 120 GiB raw about 40 GiB after adding the reserve disks.

## Consume storage and understand HA

- [Capacity and Ceph HA](docs/diataxis/explanation/ha-and-capacity.md).
- [Normal VM on node-04](docs/diataxis/how-to/native-rbd-client.md): 5 GiB RBD volume mounted at /srv/cephdata, served by host nginx.
- [k3s on node-05](docs/diataxis/how-to/k3s-rbd-client.md): Ceph-CSI, an explicit RBD StorageClass, a 5 GiB PVC and a persistent workload.

The two consumers share capacity. Pool quotas are 6 GiB native and 10 GiB k3s; they are not extra physical space. Ceph storage HA is not application HA: each client is still a single VM, and all VMs run on one host computer.

## Documentation and records

All VM-specific documentation lives in [docs/](docs/README.md), and findings live in [knowledge/okf/](knowledge/okf/README.md). Keep both current with changes. See the [architecture](docs/arc42/README.md) and [decisions](docs/adr/README.md).

| File | Purpose |
| --- | --- |
| [vagrant/Vagrantfile](vagrant/Vagrantfile) | VMs, disks and addresses |
| [inventory/vagrant.json](inventory/vagrant.json) | What the VMs reported when checked |
| [config/versions.json](config/versions.json) | Pinned images and version record |
| [config/cluster-spec.example.yaml](config/cluster-spec.example.yaml) | Hosts, labels, MON, MGR and OSD layout |
| [config/monitoring-spec.yaml](config/monitoring-spec.yaml) | Monitoring stack and HTTPS gateway |
## Current state

The three storage VMs are created, upgraded and pass the preflight. The core cluster is deployed and `HEALTH_OK`: three MONs, three MGRs and three OSDs on the `/dev/sdb` data disks, 60 GiB raw. The reserve disks are untouched. The dashboard, monitoring and an HTTPS gateway on each storage VM run, and the UI opens at `https://192.168.56.11/`, `.12/` or `.13/` from the host computer. Prometheus, Alertmanager and Grafana are under `/prometheus/`, `/alertmanager/` and `/grafana/` on each address; the first two ask for the login from `ceph orch prometheus get-credentials`. Dashboard POST requests through the gateway can fail with `503 Keep on looking`; see [ADR 0011](docs/adr/0011-gateway-template-override.md). See the [VM finding](knowledge/okf/findings/2026-11-08-vagrant-vms.md). The target configuration follows production practice where a laptop lab allows, but it is **not** production ready; [ADR 0010](docs/adr/0010-production-aligned-baseline.md) lists the gaps.

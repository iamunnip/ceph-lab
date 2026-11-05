# Ceph on three VMs

Set up a Ceph cluster on three Linux VMs, use each VM's second disk for storage, and enable the Ceph Dashboard UI. Use VirtualBox, another local hypervisor, cloud VMs or iximiuz Labs.

## Start here

1. [Prepare three VMs](docs/diataxis/how-to/vm-preflight.md).
2. [Deploy Ceph using SSH](docs/diataxis/how-to/deploy-vm-cluster.md).
3. [Enable and open the dashboard](docs/diataxis/how-to/dashboard.md).

For iximiuz Labs, use the [provider access guide](docs/diataxis/how-to/iximiuz-access.md), including `labctl expose port` for the UI. Other providers use the VM's reachable HTTPS address directly. Main guides use clearly labeled example addresses and disk paths; replace them with yours.

## Required machines

| Host | Purpose | Disks |
| --- | --- | --- |
| node-01 | MON, MGR candidate, OSD and administration | OS disk + disposable data disk |
| node-02 | MON, MGR candidate, OSD | OS disk + disposable data disk |
| node-03 | MON and OSD | OS disk + disposable data disk |

The tested setup uses 4 vCPUs, about 10 GiB RAM, a 20 GiB OS disk and an 80 GiB second disk per VM. The second disk is dedicated to Ceph; never select the OS disk. The cluster uses three replicas distributed across hosts, with a minimum of two for eligible I/O.

Three VMs are sufficient for this core track. Native clients on node-04 and k3s/CSI on node-05 are implemented extensions below. CephFS and RGW remain later work. The storage cluster itself still requires only nodes 01–03.

## Consume storage and understand HA

- [Capacity and Ceph HA](docs/diataxis/explanation/ha-and-capacity.md): 240 GiB raw, nominal 80 GiB with three replicas, approximately 76 GiB reported MAX AVAIL before client use.
- [Normal VM on node-04](docs/diataxis/how-to/native-rbd-client.md): 10 GiB RBD volume mounted at /srv/cephdata, consumed by host nginx.
- [k3s on node-05](docs/diataxis/how-to/k3s-rbd-client.md): Ceph-CSI, an explicit RBD StorageClass, a 5 GiB PVC and a persistent workload.

The two consumers share capacity. Stored-data pool budgets are 20 GiB native and 40 GiB k3s; these are not additional physical space. RBD-NBD is used for this lab's client compatibility, with the upstream CSI Alpha limitation documented. Both clients' original local data disks remain untouched.

## Documentation and records

All VM-specific documentation lives in [docs/](docs/README.md), and findings live in [knowledge/okf/](knowledge/okf/README.md). Keep both current with changes. See the [architecture](docs/arc42/README.md) and [decisions](docs/adr/README.md).

| File | Purpose |
| --- | --- |
| [inventory/example.json](inventory/example.json) | Generic three-host reference template |
| [inventory/iximiuz.json](inventory/iximiuz.json) | Observed environment, including two reserved client VMs |
| [config/versions.json](config/versions.json) | Pinned Ceph image and version record |

These JSON records are optional when following the manual walkthrough. No custom validator or command wrapper is required.

## Current deployment

The existing three-node iximiuz cluster passed its health and RADOS write/read checks. Dashboard enablement and access results are recorded in the [dashboard finding](knowledge/okf/findings/2026-11-05-dashboard.md). Previous bootstrap evidence is retained in the [original finding](knowledge/okf/findings/2026-11-05-vm-bootstrap.md).

On node-01, check the cluster directly:

```sh
sudo cephadm shell -- ceph -s
sudo cephadm shell -- ceph health detail
sudo cephadm shell -- ceph osd tree
sudo cephadm shell -- ceph mgr services
```

Expect HEALTH_OK, three MONs in quorum, three OSDs up/in, and a dashboard URL. Direct-access VMs use HTTPS; the tested iximiuz backend uses HTTP behind public HTTPS as documented in the provider guide. The guide explains each command and its expected output.

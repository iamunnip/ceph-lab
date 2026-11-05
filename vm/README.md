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
| node-01 | MON, MGR, OSD and administration | OS disk + disposable data disk |
| node-02 | MON, MGR, OSD, second administration host, monitoring and HTTPS gateway | OS disk + disposable data disk |
| node-03 | MON, MGR and OSD | OS disk + disposable data disk |

The tested setup uses 4 vCPUs, about 10 GiB RAM, a 20 GiB OS disk and an 80 GiB second disk per VM. The second disk is dedicated to Ceph as a raw device, so it has no mount point; never select the OS disk. The cluster uses three replicas distributed across hosts, with a minimum of two for eligible I/O.

Three VMs are sufficient for this core track. Native clients on node-04 and k3s/CSI on node-05 are implemented extensions below. CephFS and RGW remain later work. The storage cluster itself still requires only nodes 01–03.

## Consume storage and understand HA

- [Capacity and Ceph HA](docs/diataxis/explanation/ha-and-capacity.md): 240 GiB raw, nominal 80 GiB with three replicas, approximately 76 GiB reported MAX AVAIL before client use.
- [Normal VM on node-04](docs/diataxis/how-to/native-rbd-client.md): 10 GiB RBD volume mounted at /srv/cephdata, consumed by host nginx.
- [k3s on node-05](docs/diataxis/how-to/k3s-rbd-client.md): Ceph-CSI, an explicit RBD StorageClass, a 5 GiB PVC and a persistent workload.

The two consumers share capacity. Stored-data pool budgets are 20 GiB native and 40 GiB k3s; these are not additional physical space. RBD-NBD is used because kernel RBD failed on the tested kernel with current Ceph keys; upstream CSI labels NBD Alpha. Both clients' original local data disks remain untouched.

## Documentation and records

All VM-specific documentation lives in [docs/](docs/README.md), and findings live in [knowledge/okf/](knowledge/okf/README.md). Keep both current with changes. See the [architecture](docs/arc42/README.md) and [decisions](docs/adr/README.md).

| File | Purpose |
| --- | --- |
| [inventory/example.json](inventory/example.json) | Generic three-host reference template |
| [inventory/iximiuz.json](inventory/iximiuz.json) | Observed environment, including two reserved client VMs |
| [config/versions.json](config/versions.json) | Pinned images and version record |
| [config/cluster-spec.example.yaml](config/cluster-spec.example.yaml) | Hosts, labels, MON, MGR and OSD layout |
| [config/monitoring-spec.yaml](config/monitoring-spec.yaml) | Monitoring stack and HTTPS gateway |

These JSON records are optional when following the manual walkthrough. No custom validator or command wrapper is required.

## Current deployment

The current iximiuz deployment follows production practice where the infrastructure allows, but it is **not** production ready. [ADR 0010](docs/adr/0010-production-aligned-baseline.md) lists what is configured and each remaining gap, including the WireGuard hub on node-01, playground expiry and the absence of off-cluster backups. Measured results are in the [baseline finding](knowledge/okf/findings/2026-11-05-production-baseline.md).

On node-01, check the cluster directly:

```sh
sudo cephadm shell -- ceph -s
```

```sh
sudo cephadm shell -- ceph health detail
```

```sh
sudo cephadm shell -- ceph osd tree
```

```sh
sudo cephadm shell -- ceph orch ls
```

Expect HEALTH_OK, three MONs in quorum, three MGRs, three OSDs up and in, and all services at their expected counts. The [deployment guide](docs/diataxis/how-to/deploy-vm-cluster.md#verification-and-recovery) explains the output.

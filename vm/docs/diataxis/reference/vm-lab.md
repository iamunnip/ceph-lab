# VM lab reference

The reusable lab requires three reachable Linux VMs, three dedicated data disks, unique hostnames, sudo-enabled SSH access and sufficient resources. The [generic inventory template](../../../inventory/example.json) uses example addresses; replace them. The observed values below describe our iximiuz deployment. See [ADR 0005](../../adr/0005-provider-independent-guides.md) for the portability policy.

Status: topology planned below; actual [host inventory](../../../inventory/iximiuz.json) was collected on 2026-11-05. Logical ceph-1/2/3 map to node-01/02/03; clients map to node-04/05. See [deployment evidence](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md).

| Logical name | Role | Additional storage | Planned daemons/software |
| --- | --- | --- | --- |
| ceph-1 | Storage and initial administration | 80 GB | MON, MGR active candidate, OSD |
| ceph-2 | Storage | 80 GB | MON, MGR standby candidate, OSD |
| ceph-3 | Storage | 80 GB | MON, OSD |
| client-native (implemented extension) | node-04 Linux storage consumer | Local disk unused by Ceph | 10 GiB RBD-NBD/ext4 mount and host nginx |
| client-k3s (implemented extension) | node-05 single-node Kubernetes | Local disk unused by Ceph | k3s, Ceph-CSI, 5 GiB RWO PVC and nginx |

CephFS adds two MDS daemons on different storage hosts (one active, one standby), subject to measured RAM availability. RGW is optional and has no HA commitment.

Record actual hostnames, reachable Ceph IPs, OS/kernel, vCPU/RAM, root disk size/free space, data disk stable identifier and capacity, container runtime, SSH user, and time sync state during preflight. Logical names are labels, not claims about existing VM names. Keep private access details and credentials outside Git.

## Storage and failure policy

- One explicitly identified whole data device per storage VM; root devices are excluded.
- Replicated pools: `size=3`, `min_size=2`, CRUSH failure domain `host`.
- Verified raw capacity: 240 GiB; nominal replicated capacity: 80 GiB before metadata, allocation overhead and operational headroom. Ceph initially reported about 76 GiB shared MAX AVAIL.
- Keep test data small and establish a measured capacity budget before recovery tests. Do not fill the cluster to its full/nearfull thresholds.
- One storage-host outage should retain two-monitor quorum and allow eligible replicated-pool I/O while degraded. With only two remaining OSDs, three replicas cannot be restored until the third host returns.
- Two storage-host outages lose MON quorum and do not meet the planned pool write policy. No continued-service guarantee is made.

## Network contract

Use an existing routed network between the three VMs and browser host. WireGuard is not required. In our observed iximiuz deployment, WireGuard is already configured. Reuse existing addresses and routes; do not configure peers, keys, hubs, or spokes. Verify all storage hosts and clients can reach the advertised Ceph endpoints, including direct client-to-OSD traffic through the existing routing.

In that deployment, use the existing WireGuard network as the Ceph public network initially; no separate replication network is planned. Confirm monitor ports (normally TCP 3300 and legacy 6789 if enabled) and the selected release's daemon port range (commonly TCP 6800–7568). Inspect actual listeners and firewall rules. Record effective MTU and reachability under larger transfers. Do not broadly disable firewalls.

A hub outage may isolate multiple nodes at once. The five VMs are not proof of independent underlying hardware failure domains. Single-node k3s is a storage integration test, not an HA Kubernetes cluster.

## Version contract

Selected Ceph/cephadm 20.2.5 and Podman 5.7.0; see [version pins](../../../config/versions.json). Client versions: k3s v1.36.6+k3s1, Ceph-CSI 3.18.2, Helm v3.19.0. Native mapping uses the pinned Ceph 20.2.5 container; CSI userspace is Ceph 20.2.4. Record Ceph release and image digest, cephadm version/source, host OS/kernel/runtime, k3s/Kubernetes version, Ceph-CSI release/chart and image digests, Helm and client versions. Check supported combinations in the selected release documentation. Upstream `latest` documentation is orientation only and may describe development releases.

## Sources

- [Cephadm deployment requirements](https://docs.ceph.com/en/latest/cephadm/install/)
- [iximiuz Labs playground building blocks](https://labs.iximiuz.com/docs/custom-playgrounds/building-blocks)
- [Ceph-CSI project and compatibility information](https://github.com/ceph/ceph-csi)
- [Ceph RBD with CSI](https://docs.ceph.com/en/latest/csi/rbd/)

## Dashboard

The UI is part of the core three-VM setup. It runs in the active MGR with a dashboard administrator. Direct VM access uses HTTPS on 8443 with a self-signed lab certificate. The observed iximiuz setup uses an approved HTTP backend on 18080 behind a public HTTPS URL because its ingress could not negotiate the Ceph backend TLS. Follow the [dashboard guide](../how-to/dashboard.md). On VirtualBox or other reachable VMs, open the active manager's address directly. On iximiuz, use `labctl expose port` as described in the [provider guide](../how-to/iximiuz-access.md). Do not expose MON or OSD ports publicly to open the UI.

## Client capacity and HA evidence

[HA/capacity explanation](../explanation/ha-and-capacity.md) covers shared pool space, thin provisioning and the 60 GiB combined logical budget. The [client finding](../../../knowledge/okf/findings/2026-11-05-client-storage.md) records persistence tests and successful reads/writes on both consumers while node-03 MON/OSD were unavailable. Full VM power loss remains untested.

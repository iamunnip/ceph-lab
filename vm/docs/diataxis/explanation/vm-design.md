# VM design rationale

Cephadm keeps the first learning track focused on Ceph rather than Kubernetes scheduling. Three storage hosts let the lab explore monitor quorum and host-level replica placement. One OSD per host makes failures easy to observe, but gives no spare host for restoring a third replica during a host outage.

RBD comes first because it exposes image lifecycle, client authentication and block-device persistence. CephFS adds metadata services and shared filesystem behavior. The native client and single-node k3s client consume the same external Ceph cluster, demonstrating that CSI integrates Kubernetes with storage independently of where Ceph runs.

In our iximiuz example, WireGuard is an existing infrastructure dependency. Other deployments can use a reachable LAN, routed private network or VPN without WireGuard. Routing through a hub affects both failure behavior and throughput. Client access requires reachable OSDs as well as monitors; reaching a MON alone is insufficient. Network and storage failure domains must be described separately.

Start with replicated pools, not erasure coding, to keep the first experiments understandable and compatible with the three-host capacity limit. Use scoped CephX identities for native services and CSI; admin credentials are for administration only. The lab's limited size and shared infrastructure do not establish production readiness or HA guarantees for applications.

See the [accepted decisions](../../adr/README.md) and [VM reference](../reference/vm-lab.md).

The core walkthrough now ends with three storage VMs and the Ceph dashboard. Native RBD services and k3s/CSI are implemented client extensions; CephFS and RGW remain later exercises. The core cluster can still be learned with only three VMs; two more are used for the client exercises. All VM-specific documentation and knowledge live under this track. See [ADR 0007](../../adr/0007-vm-owned-docs-and-dashboard.md).

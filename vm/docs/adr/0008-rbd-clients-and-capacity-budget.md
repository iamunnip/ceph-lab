# 0008: Consume replicated RBD from native and k3s clients

Status: accepted and implemented. Date: 2026-11-05.

## Context

The three-host Ceph cluster has one 80 GiB OSD per host. The user requests focus on Ceph HA, usable capacity and storage consumption from native node-04 and k3s node-05, leaving existing networking unchanged.

## Decision

Keep storage on nodes 01–03. Use three host-separated replicas with pool size 3/min_size 2. Add separate RBD pools and scoped CephX identities for each consumer: native-rbd/client.native and k3s-rbd/client.k3s. Set logical stored-data quotas of 20 GiB and 40 GiB, a combined 60 GiB budget below nominal 80 GiB usable capacity. Quotas are not physical reservations and do not prevent thin-provisioned allocation beyond available capacity.

Node-04 uses a 10 GiB ext4 RBD image at /srv/cephdata and host nginx. Node-05 uses single-node k3s v1.36.6+k3s1 with Ceph-CSI chart 3.18.2 and a separate 5 GiB RWO PVC. StorageClass ceph-rbd is explicit and uses Retain; the client VM disks remain untouched. The VM cluster remains managed by cephadm; no second cluster or Rook is deployed.

Use matching Ceph 20.2.5 userspace in a pinned container for native mapping and the CSI release's Ceph 20.2.4 userspace for Kubernetes. The host's older Ceph package failed to parse the newer cluster credential, so use RBD-NBD instead of relaxing authentication or assuming kernel client compatibility. CSI upstream labels its NBD path Alpha: this is a measured lab compatibility choice, not a production recommendation. Pin exact artifacts in [versions](../../config/versions.json).

Keep native mapping, filesystem and nginx dependencies explicit in systemd. Disconnect NBD using nbd-client during shutdown before stopping the mapper container; stopping the process alone left a stale netlink connection during testing. Require the Ceph mount before serving data.

## Consequences

240 GiB raw supports nominal 80 GiB with three replicas; observed MAX AVAIL is about 76 GiB before client usage. All pools share that budget. RBD/PVC requested size is not the same as actual disk use. Ordinary ext4 RBD images are not shared read-write between the two clients.

Single-host Ceph service failure can leave the two replicas available for I/O. With only three storage hosts, the third replica cannot be rebuilt until that host or replacement storage returns. This does not make either client VM or single-node k3s highly available.

## Evidence

The [client finding](../../knowledge/okf/findings/2026-11-05-client-storage.md) records actual mounts, PVC provisioning, checksums, permission checks and the controlled node-03 MON/OSD outage. See [HA and capacity](../diataxis/explanation/ha-and-capacity.md), [native client](../diataxis/how-to/native-rbd-client.md) and [k3s client](../diataxis/how-to/k3s-rbd-client.md) for reader instructions.

This implements the RBD client progression from ADR 0003. The three-host core setup in ADR 0007 remains valid; these clients are now implemented extensions.

## Follow-up on 2026-11-05 redeployment

[ADR 0010](0010-production-aligned-baseline.md) keeps RBD-NBD after testing kernel RBD, which failed on kernel 6.1.167 with `libceph: secret too big 32`. It also enables the full image feature set and sets target size ratios.

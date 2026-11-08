# 0008: Consume replicated RBD from native and k3s clients

Status: accepted; implementation planned. Date: 2026-11-05, revised 2026-11-08.

## Context

The three storage hosts start with one 20 GiB OSD each: 60 GiB raw and about 20 GiB usable with three copies. The reserve disks later double that. node-04 and node-05 should consume Ceph storage.

## Decision

Keep storage on nodes 01 to 03. Use three host-separated replicas with pool size 3/min_size 2. Add separate RBD pools and scoped CephX identities for each consumer: native-rbd/client.native and k3s-rbd/client.k3s. Set stored-data quotas of 6 GiB and 10 GiB, 16 GiB in total, below the roughly 20 GiB usable at first. Quotas are not reservations and do not prevent thin-provisioned allocation beyond available capacity.

node-04 uses a 5 GiB ext4 RBD image at /srv/cephdata and host nginx. node-05 uses single-node k3s v1.36.6+k3s1 with Ceph-CSI chart 3.18.2 and a 5 GiB RWO PVC. StorageClass ceph-rbd is explicit and uses Retain. The cluster remains managed by cephadm; no second cluster or Rook is deployed.

Try kernel RBD first, because it is the production path. Older kernels reject the 32-byte keys of this Ceph release with `libceph: secret too big 32`; the upgraded kernel 7.0.0-43 is untested. If kernel RBD fails, use RBD-NBD with matching Ceph 20.2.5 userspace in a pinned container for native mapping, and the CSI release's Ceph 20.2.4 userspace for Kubernetes. Never relax authentication to suit an older client. CSI upstream labels its NBD path Alpha. Pin exact artifacts in [versions](../../config/versions.json).

Keep native mapping, filesystem and nginx dependencies explicit in systemd. Disconnect NBD using nbd-client during shutdown before stopping the mapper container. Require the Ceph mount before serving data.

## Consequences

With 16 GiB of quotas on about 20 GiB usable, the clients leave little headroom until the reserve disks are added. All pools share the same space. RBD and PVC sizes are not the same as actual disk use. Ordinary ext4 RBD images are not shared read-write between the two clients.

Single-host Ceph service failure can leave two replicas available for I/O. The third replica cannot be rebuilt until that host returns. This does not make either client VM or single-node k3s highly available.

See [HA and capacity](../diataxis/explanation/ha-and-capacity.md), [native client](../diataxis/how-to/native-rbd-client.md) and [k3s client](../diataxis/how-to/k3s-rbd-client.md). This implements the RBD client progression from ADR 0003.

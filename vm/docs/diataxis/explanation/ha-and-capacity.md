# Ceph HA and usable capacity

Ceph has no single switch called “HA mode.” Our HA design combines three MONs, one active and two standby MGRs, and three copies of pool data distributed across three storage hosts.

## What one failure means

| Condition | Expected Ceph behavior |
| --- | --- |
| All three storage hosts healthy | MON quorum and three data copies; clean PGs |
| One storage host's Ceph services unavailable | Two MONs retain quorum; replicated-pool I/O can continue with two available copies after failure detection/peering |
| One data disk/OSD unavailable | Data access can continue with two available copies; replication is degraded |
| Active MGR unavailable | A standby takes over management; the gateway URL stays the same |
| Two storage hosts unavailable | MON quorum lost and pool min_size not met; this setup cannot provide normal storage service |

`size=3` requests three copies. `min_size=2` requires at least two copies for pool I/O; it is not a desired replica count. CRUSH must place those copies on separate hosts. Inspect these settings on node-01:

```sh
sudo cephadm shell -- ceph quorum_status
sudo cephadm shell -- ceph osd tree
sudo cephadm shell -- ceph osd pool ls detail
sudo cephadm shell -- ceph osd crush rule dump
```

With only three OSD hosts, Ceph cannot rebuild a third host-separated copy until the missing host returns or replacement storage is added. HA can include a brief pause while clients detect failure and PGs peer. Three replicas do not replace an independent backup.

## Where the capacity goes

| Item | Capacity |
| --- | --- |
| Storage VM OS disk | 20 GiB each; excluded from OSD capacity |
| Storage VM data disk | 80 GiB each |
| Total raw Ceph space | 3 × 80 = 240 GiB |
| Nominal logical space with three replicas | 240 ÷ 3 = 80 GiB |
| Observed initial Ceph MAX AVAIL | About 76 GiB; changes with usage and placement |
| Planned combined client data budget | 60 GiB logical, leaving headroom |

A 1 GiB payload usually needs roughly 3 GiB of raw data space before metadata/allocation overhead. Filesystem free space is smaller again because ext4 has its own metadata. GB and GiB are different units: 80 GiB is about 85.9 decimal GB; use the units shown by the tools.

Pools share the same disks. If native-rbd and k3s-rbd both show 76 GiB MAX AVAIL, that does not mean 152 GiB total. Our stored-data pool quotas are 20 GiB native and 40 GiB k3s; they do not reserve that space. Other pools and snapshots still consume storage.

The 10 GiB native image and 5 GiB Kubernetes PVC are thin-provisioned allocations. Creating them does not immediately write 45 GiB raw. Actual writes, filesystem metadata and retained snapshots consume space over time. Avoid assuming the sum of image/PVC sizes is an accurate measure of disk usage.

Check on node-01:

```sh
sudo cephadm shell -- ceph df
sudo cephadm shell -- ceph osd df
sudo cephadm shell -- rbd du --pool native-rbd
sudo cephadm shell -- rbd du --pool k3s-rbd
```

## The two consumers

Node-04 maps its own RBD image, mounts ext4 at /srv/cephdata and runs a native nginx service using that mount. Node-05 runs single-node k3s; Ceph-CSI creates and mounts separate images for PVCs. The clients' local second disks are not part of the cluster.

Both clients get Ceph's replicated storage, but that does not make either client VM or the single k3s server highly available. Ordinary ext4 RBD volumes must not be mounted read-write by multiple independent clients. Use separate RBD images; shared filesystem consumption is a later CephFS exercise.

## HA depends on the network too

Ceph can only be as available as the network between its hosts. In the iximiuz example, node-01 is the WireGuard hub, so all traffic between other nodes passes through it. Rebooting node-01 also cut node-02 off from node-03, even though both were healthy, and one client write stalled for 27 s before recovering without errors. A production cluster needs storage hosts connected without a single shared relay. With a different network, this limit does not apply.

Stopping Ceph on spoke node-03 behaved as the table above predicts: both clients wrote in under 0.25 s while PGs were undersized, and health returned 13 s after restoring services. See the [production baseline finding](../../../knowledge/okf/findings/2026-11-05-production-baseline.md).

See [Ceph pool settings](https://docs.ceph.com/en/tentacle/rados/operations/pools/) and [monitor quorum](https://docs.ceph.com/en/tentacle/rados/configuration/mon-config-ref/). Measured results are in the [client finding](../../../knowledge/okf/findings/2026-11-05-client-storage.md).

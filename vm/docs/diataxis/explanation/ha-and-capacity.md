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
| Storage VM OS disk | 20 GiB each; never part of Ceph |
| Storage VM data disk | 20 GiB each, the first OSD |
| Storage VM reserve disk | 20 GiB each, empty until [added](../how-to/add-reserve-disks.md) |
| Raw Ceph space at first | 3 × 20 = 60 GiB |
| Usable with three copies at first | 60 ÷ 3 = 20 GiB, a little less after overhead |
| Raw Ceph space after adding the reserve disks | 6 × 20 = 120 GiB |
| Usable with three copies after that | 120 ÷ 3 = 40 GiB |
| Planned client data quotas | 6 GiB native and 10 GiB k3s, 16 GiB in total |

A 1 GiB payload usually needs roughly 3 GiB of raw space before metadata and allocation overhead. Filesystem free space is smaller again because ext4 has its own metadata. GB and GiB are different units: 20 GiB is about 21.5 decimal GB; use the units shown by the tools.

The small disks are on purpose. With about 20 GiB usable, the cluster reaches Ceph's nearfull warning quickly, which is what the [reserve disk exercise](../how-to/add-reserve-disks.md) needs. The client quotas leave little headroom until the reserve disks are added.

Pools share the same disks. If native-rbd and k3s-rbd both show 18 GiB MAX AVAIL, that does not mean 36 GiB in total. Quotas limit what a pool may store; they do not reserve that space. Other pools and snapshots still consume storage.

The 5 GiB native image and 5 GiB Kubernetes PVC are thin-provisioned. Creating them does not immediately use 30 GiB raw. Actual writes, filesystem metadata and retained snapshots consume space over time. The sum of image and PVC sizes is not a measure of disk usage.

Check on node-01:

```sh
sudo cephadm shell -- ceph df
sudo cephadm shell -- ceph osd df
sudo cephadm shell -- rbd du --pool native-rbd
sudo cephadm shell -- rbd du --pool k3s-rbd
```

## The two consumers

Node-04 maps its own RBD image, mounts ext4 at /srv/cephdata and runs a native nginx service using that mount. Node-05 runs single-node k3s; Ceph-CSI creates and mounts separate images for PVCs. The client VMs have no extra disks; all their persistent data lives in Ceph.

Both clients get Ceph's replicated storage, but that does not make either client VM or the single k3s server highly available. Ordinary ext4 RBD volumes must not be mounted read-write by multiple independent clients. Use separate RBD images; shared filesystem consumption is a later CephFS exercise.

## HA depends on the host too

All five VMs run on one computer and share its disk, CPU and memory. Ceph can survive losing one VM, and you can test that by stopping one with `vagrant halt node-03`. But if the host computer stops or its disk fails, every VM and every copy goes with it. Three replicas inside one laptop teach how Ceph behaves; they do not protect data.

The VMs talk over the VirtualBox host-only network, a software switch inside the host. It has no physical failure modes of its own, so network failure tests need deliberate steps, such as bringing down `enp0s8` inside one VM.

See [Ceph pool settings](https://docs.ceph.com/en/tentacle/rados/operations/pools/) and [monitor quorum](https://docs.ceph.com/en/tentacle/rados/configuration/mon-config-ref/).

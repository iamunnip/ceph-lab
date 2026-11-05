# Ceph HA, capacity and client consumption, 2026-11-05

Status: both clients deployed; persistence and single-host Ceph-service HA checks passed; cluster restored to HEALTH_OK.

## Scope

The user requested focus on Ceph HA, 80 GiB-per-node capacity, usable space, and consumption by native node-04 and k3s node-05. Existing networking is unchanged. The storage cluster remains on nodes 01–03; client VMs are consumers, not additional OSD hosts.

## Capacity and policy

Observed `ceph df`: 240 GiB raw, about 76 GiB MAX AVAIL before client deployment. Three replicas mean a nominal 80 GiB logical capacity before overhead/headroom. MAX AVAIL shown for multiple pools backed by the same OSDs is shared, not additive. Data stored on either client consumes that same capacity. Local 80 GiB disks on clients are not added to Ceph.

Created native-rbd and k3s-rbd, both explicitly size 3/min_size 2 with host-level CRUSH rule. Set logical pool quotas to 20 GiB and 40 GiB respectively, as a conservative combined 60 GiB budget. Quotas cap stored pool data rather than reserve physical space or cap thin-provisioned image size. Snapshots, metadata and other pools still consume capacity.

Native image: native-rbd/native-data, 10 GiB, layering plus exclusive-lock. k3s PVC: ceph-demo/web-data, 5 GiB RWO, ceph-rbd StorageClass, Retain reclaim policy. Images are thin-provisioned: requested size differs from physically used capacity.

## Compatibility finding

Ubuntu package ceph-common=20.2.0-0ubuntu2.1 could not parse the cluster's newer CephX credential. The failed client printed the unused native key in its error; revoked/replaced it before use. No keys are committed. Matching Ceph 20.2.5 tools in the pinned cluster image successfully created the image. Chose userspace RBD-NBD for mapping rather than weakening authentication or assuming the 6.1.167 kernel supports newer key types. NBD module/device availability confirmed on both clients.

Ceph-CSI's pinned [RBD-NBD documentation](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/docs/design/proposals/rbd-nbd.md) labels this path Alpha. It is a lab compatibility choice, not a production recommendation. The native mapper is a privileged, systemd-managed Ceph container; nginx itself runs as an ordinary host service.

## Version selection

k3s v1.36.6+k3s1 and Ceph-CSI chart/app 3.18.2. The [CSI release matrix](https://raw.githubusercontent.com/ceph/ceph-csi/v3.18.2/README.md) includes Kubernetes 1.36. Helm v3.19.0 archive checksum verified against its official SHA-256 file. CSI chart archive SHA-256: 62e0c93615a63c7a4cc91211011f378c0318d00ba0e034be5c817c6f52dba897.

k3s uses node-05 and its existing 10.8.0.5 address; default local-storage, Traefik and ServiceLB are disabled. Actual kubelet directory is /var/lib/kubelet. CSI namespace permits privileged node-plugin pods, provisioner has one replica for this one-node lab, and node-plugin updates use OnDelete as recommended for NBD mappings. Scoped client.k3s has access only to k3s-rbd; client.native only to native-rbd.

## Current evidence

k3s node-05 Ready; CSI provisioner 7/7 and node-plugin 3/3 Running. PVC Bound, capacity 5 GiB, RWO, explicit ceph-rbd class. nginx workload became Ready. Native exclusive mapping initially failed because the image had only layering; enabling exclusive-lock resolved that prerequisite.

## Native persistence and access checks

Mounted the new 10 GiB image as ext4 at /srv/cephdata. `df` reported about 9.8 GiB filesystem capacity. Host nginx serves it on 18081, with a required mount dependency. Both the initial request and the final remount request returned the expected page.

Stopping the initial mapper process without an explicit kernel disconnect left stale NBD state. This caused the first remap test to fail with Object busy; no reformat was performed. Installed nbd-client, disconnected the unmounted stale session, and changed the unit's ExecStop to `nbd-client -d /dev/nbd0` before stopping its container. Repeated the complete nginx-stop, unmount, mapper-stop, mapper-start, remount and nginx-start sequence successfully. Original file checksum before and after:

```text
e046fa091671d1fa56fc5fcbafef199eac00f088145e1cea2b8f259a3d793d87
```

The functioning native identity listed its own native-data image and was explicitly denied access to k3s-rbd with an expected permissions error. An earlier check while the mapper container was unavailable was inconclusive and was replaced by this real permissions test.

## Kubernetes persistence

The 5 GiB PVC is Bound using ceph-rbd and Retain. The nginx deployment became Ready. Wrote a unique marker, deleted the pod, waited for the replacement to become Ready, and verified the marker and SHA-256 were unchanged:

```text
92cdd7843fb29ad713ce5cace2e8647a7a90874ab2857c936107aafe19c6a6e3
```

Then pinned the nginx deployment to its observed immutable digest and verified both the original marker and later HA marker survived that rollout. CSI image digest and its Ceph userspace version (20.2.4) are recorded in versions.json. The chart checksum matches the official index.

## Controlled Ceph HA experiment

From a healthy baseline, stopped node-03's MON and OSD.2 services. A temporary 600-second safety-restore timer was scheduled before stopping services, and canceled after explicit restoration. This simulates loss of one host's Ceph services, not full VM power loss or network failure.

Observed while down: MON quorum node-01/node-02, two OSDs up, degraded/undersized PGs. Both consumers wrote a new file and synced it while this degraded state was confirmed. Native nginx and the k3s workload both continued serving the original data, with unchanged original checksums. The newly written markers were `native-ha-write-during-node03-outage` and `k3s-ha-write-during-node03-outage`.

Restarted node-03 MON/OSD; both systemd units returned active. Watched recovery progress, then verified HEALTH_OK, all three MONs in quorum, three OSDs up/in, and 97 active+clean PGs. A data-redundancy warning during recovery cleared without changing replica policy. Pool settings remain size 3/min_size 2 and CRUSH chooseleaf type host.

Final sample: about 781 MiB raw used, 239 GiB raw available, with 240 GiB total; this changes as client filesystems write metadata. Both client pools still reported around 76 GiB shared MAX AVAIL. Quota commands confirmed 20 GiB native and 40 GiB k3s logical limits. Reported estimates are rounded and are not separate capacity allocations per pool.

## Limitations and remaining validation

Full VM reboot/power-loss tests, disk replacement, quota-exhaustion behavior and application failover are not tested. A single native service VM and a single k3s node are not HA applications. NBD is the measured lab compatibility path and the upstream CSI NBD status is Alpha. CephFS/RGW, snapshot backup/restore and upgrades remain separate exercises. Clients' local second disks, storage root disks and existing network configuration were untouched.


Final client checks: native map, mount and nginx units all active; native outage marker readable. k3s workload Ready and 5 GiB PVC Bound after digest pinning; Kubernetes outage marker readable. Both original /dev/vdb client disks remain ext4 at /data. All local links resolved across 31 Markdown files, JSON records parsed, whitespace checks passed, and no literal Ceph credential values were found in documents/configuration assets.

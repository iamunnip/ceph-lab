# 0010: Align the VM deployment with production practice

Status: accepted; deployed and tested end to end on 2026-11-09, see the [finding](../findings/2026-11-09-end-to-end.md). Date: 2026-11-05, revised 2026-11-08.

## Context

The user asked for production-level, recommended configuration where the lab allows: node-01 to node-03 as the Ceph cluster, node-04 as a VM consumer and node-05 as a k3s consumer using storage drivers.

## Decision

**Cluster layout.** Apply a declarative [cluster spec](../../../configs/podman/cluster-spec.example.yaml) with host labels instead of one-off placement commands. Run three MGRs. Put the `_admin` label and the cephadm binary on two hosts, so losing node-01 does not remove all administration. Select OSD disks by explicit path in an OSD service spec.

**Monitoring.** Deploy Prometheus, Alertmanager, Grafana, node-exporter and ceph-exporter through a [monitoring spec](../../../configs/podman/monitoring-spec.yaml). Pin every image by digest. Cap Prometheus at 2 GB and 15 days to protect the 20 GB OS disks.

**Dashboard access.** Use cephadm's `mgmt-gateway` for HTTPS on port 443 of every storage VM (label `gateway` on all three hosts, first only node-02). It routes to the active manager, so the URL survives manager failover, and it accepts TLS 1.2 and 1.3. Open it from the host computer at `https://192.168.56.12/`. The dashboard's own certificate is issued by the cephadm root CA with `ceph orch certmgr generate-certificates dashboard`, not by `create-self-signed-cert`, because the gateway's internal port requires client certificates from that CA. Never use the dashboard's **Add storage** wizard here: it creates an OSD service for all spinning disks and takes the reserve disks.

**Pools.** Size 3, min_size 2, host failure domain and the quotas from ADR 0008. Set `target_size_ratio` so PGs are allocated before data arrives. Keep `mon_allow_pool_delete` false, except briefly to remove a test pool.

**Client mapping.** Kernel RBD is the recommended production path and is tried first. RBD-NBD is a documented fallback only if the kernel rejects current keys. Enable the full librbd image feature set on both clients.

**k3s.** Configure through `/etc/rancher/k3s/config.yaml`. Use embedded etcd for scheduled snapshots, and encrypt Secrets at rest because one of them holds the Ceph key. Keep the Retain reclaim policy and the explicit StorageClass.

**Native client.** Restart the mapper on failure, load NBD at boot if it is used, and enable `fstrim.timer` so deleted blocks return to Ceph.

**Unchanged on purpose.** One public network and no cluster network, since the VMs have one usable network. Default msgr2 mode without on-wire encryption, since the host-only network does not leave the host computer.

## Consequences

This is production-aligned configuration on non-production infrastructure. It does not make the lab production ready. The known gaps:

| Gap | Why it matters | What production needs |
| --- | --- | --- |
| One host computer | A host failure loses every VM and every copy | Separate physical servers, power and disks |
| Shared host disk | All OSDs write to the same physical disk | Dedicated disks per OSD |
| Exactly three hosts | Lost replicas cannot rebuild until the host returns | Four or more OSD hosts, several OSDs per host |
| Small memory | 6 GB per VM; OSD target 1.5 GiB instead of the 4 GiB default | Sized RAM per OSD and daemon |
| No off-cluster backup | Replicas and snapshots are not backups | Independent backup and tested restore |
| Alertmanager has no receivers | Alerts fire but nobody is notified | Configured notification route |
| Self-signed certificate | Browser warning, no identity assurance | Certificate from a trusted CA |
| Single k3s node and single native VM | Storage is HA, applications are not | Several servers and application failover |
| No host firewall rules | Relies on the host-only network staying private | Rules limiting Ceph ports to known networks |
| Disks report as rotational | OSDs get class `hdd` whatever the real media | Known hardware and correct device classes |

Since 2026-11-08 a gateway runs on each of the three storage VMs, so the UI stays reachable on another address if one VM is down. `virtual_ip` is not configured, so switching addresses is manual.

## Relation to earlier decisions

Refines ADR 0002 and ADR 0008, and the dashboard decision in ADR 0007.

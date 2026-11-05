# 0010: Align the VM deployment with production practice

Status: accepted and implemented. Date: 2026-11-05.

## Context

The user asked for a redeployment on five new playgrounds using production-level, recommended configuration: node-01 to node-03 as the Ceph cluster, node-04 as a VM consumer and node-05 as a k3s consumer using storage drivers. The earlier deployment made several lab-only choices, some of which are now testable or replaceable.

## Decision

**Cluster layout.** Apply a declarative [cluster spec](../../config/cluster-spec.example.yaml) with host labels instead of one-off placement commands. Run three MGRs instead of two. Put the `_admin` label and the cephadm binary on two hosts, so losing node-01 does not remove all administration. Select OSD disks by explicit path in an OSD service spec.

**Monitoring.** Deploy Prometheus, Alertmanager, Grafana, node-exporter and ceph-exporter through a [monitoring spec](../../config/monitoring-spec.yaml), replacing the deferral in ADR 0004. Pin every image by digest. Cap Prometheus at 2 GB and 15 days because root disks are 20 GiB.

**Dashboard access.** Use cephadm's `mgmt-gateway` for HTTPS on port 443. It routes to the active manager, so the URL survives manager failover, and it accepts TLS 1.2 and 1.3. This supersedes the plain HTTP backend on 18080 and the "no additional proxy" rule in ADR 0007, because the gateway is a Ceph-managed service that removes the need for both the workaround and re-exposure after failover. On iximiuz, expose it with `labctl expose port <playground> 443 --https` without `--public`, so an iximiuz login is required in front of the dashboard login.

**Pools.** Keep size 3, min_size 2, host failure domain and the 20 GiB and 40 GiB quotas from ADR 0008. Set `target_size_ratio` so PGs are allocated before data arrives. Remove the retained `lab-smoke` pool from the deployment. Keep `mon_allow_pool_delete` false.

**Client mapping.** Kernel RBD is the recommended production path and was tested first. On kernel 6.1.167 it failed with `libceph: secret too big 32`, because the kernel cannot load the 32-byte keys issued by this Ceph release. Weakening the credential is not allowed, and the provider controls the kernel. RBD-NBD therefore remains, as a documented gap rather than a preference. Enable the full librbd image feature set on both clients. A future deployment on a kernel that accepts these keys should move to kernel RBD and retest.

**k3s.** Configure through `/etc/rancher/k3s/config.yaml`. Use embedded etcd for scheduled snapshots, and encrypt Secrets at rest because one of them holds the Ceph key. Keep the Retain reclaim policy and the explicit StorageClass.

**Native client.** Restart the mapper on failure, load NBD at boot and enable `fstrim.timer` so deleted blocks return to Ceph.

**Unchanged on purpose.** One public network and no cluster network, since only `mesh0` has unique addresses. Default msgr2 mode, since WireGuard already encrypts traffic between hosts. WireGuard itself is not changed.

## Consequences

Failure and reboot behavior is now measured, see the [finding](../../knowledge/okf/findings/2026-11-05-production-baseline.md). Root disk use after deployment is 20 to 28 percent, with Prometheus retention still to grow on node-02. The gateway host is a single point for UI access only; `virtual_ip` needs a network this provider does not offer.

This is production-aligned configuration on non-production infrastructure. It does not make the lab production ready. The remaining gaps are:

| Gap | Why it matters | What production needs |
| --- | --- | --- |
| WireGuard hub on node-01 | Measured: a node-01 reboot also cut node-02 from node-03 | Independent network paths between storage hosts |
| Network quality | MTU 1280, up to about 30 ms RTT, shared by all traffic | Low-latency LAN, ideally 10 GbE or better |
| Exactly three hosts | Lost replicas cannot rebuild until the host returns | Four or more OSD hosts, several OSDs per host |
| RBD-NBD | Upstream CSI labels it Alpha, and it depends on a userspace process | A kernel that accepts current keys, then kernel RBD |
| Playground lifetime | Playgrounds expire, about 24 hours here | Durable hosts |
| No off-cluster backup | Replicas and snapshots are not backups | Independent backup and tested restore |
| Alertmanager has no receivers | Alerts fire but nobody is notified | Configured notification route |
| Self-signed certificate | Browser warning, no identity assurance | Certificate from a trusted CA |
| Single k3s node and single native VM | Storage is HA, applications are not | Several servers and application failover |
| No host firewall rules | Relies on provider isolation | Rules limiting Ceph ports to known networks |
| Disks report as rotational | OSDs get class `hdd` whatever the real media | Known hardware and correct device classes |

## Relation to earlier decisions

Refines ADR 0002 and ADR 0008. Supersedes the monitoring deferral and `lab-smoke` retention in ADR 0004, and the dashboard backend, proxy and `--public` choices in ADR 0007. The RBD-NBD decision in ADR 0008 stands, now with a kernel test behind it.

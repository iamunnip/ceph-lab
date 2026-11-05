# VM validation matrix

Current results are in the [production baseline finding](../../../knowledge/okf/findings/2026-11-05-production-baseline.md). Passed on 2026-11-05: baseline, dashboard through the gateway except an interactive browser session, native and k3s persistence across client reboot, cross-pool denial, MGR failover, node-03 reboot, sustained node-03 service outage and a hub reboot. Earlier evidence is in the [client finding](../../../knowledge/okf/findings/2026-11-05-client-storage.md). Power loss, CephFS, teardown and later cases remain pending. Record date, exact versions, inputs, sanitized commands/output, expected versus observed behavior and recovery outcome in `vm/knowledge/okf/`.

| Case | Required evidence |
| --- | --- |
| Preflight | Three storage hosts inventoried; optional two clients inventoried for consumption exercises; three data disks positively identified; resource/network requirements satisfied |
| Baseline | Three MONs in quorum; MGR active/standby; three OSDs up/in; expected host placement; clean PGs and explained health state |
| Dashboard | Gateway HTTPS listener, administrator authentication, required password change, URL unchanged after MGR failover, direct or provider URL access |
| Monitoring | Prometheus, Alertmanager, Grafana and exporters running at expected counts, on pinned digests |
| Kernel client | Real kernel map attempted before choosing NBD; retest on any new kernel or Ceph release |
| Native RBD (extension) | Known checksums preserved after remount, service restart and client reboot; mount ordering prevents service writing into an unmounted local directory |
| k3s RBD | Bound PVC and correct CSI driver/pool; checksums survive pod deletion/recreation and client VM reboot |
| CephFS | Active/standby MDS; shared writes visible between native client and k3s; RWX works across two pods |
| OSD outage | Stop one OSD; record degraded state, continued eligible I/O and checksums; restore and wait for clean PGs |
| Storage-host outage | Stop one host including its MON and sample cluster state during the outage, not only after it; verify quorum, degraded I/O and checksums; restore to healthy baseline |
| Client outage | Restart client; verify remount/reattach, workload readiness and persisted data |
| Network interruption | In a controlled disposable environment, interrupt one selected endpoint without changing WireGuard configuration; record affected roles, observed I/O, recovery and management access |
| Negative credentials | Scoped client cannot administer the cluster or access unrelated pools/filesystems |
| Reproducibility | Repeat supported automation stages without accidental rebootstrap, duplicate resources or data loss |
| Teardown | Explicit target list, credential cleanup and documented retained/destroyed resources |

Run failures one at a time from a healthy baseline. Define timeout, stop conditions and recovery steps before each experiment. Never treat a snapshot in the same cluster as an independent backup. Performance results are illustrative for this shared VM/network lab and must include dataset size, client concurrency, replication settings and recovery state.

# VM validation matrix

Cluster baseline, RADOS write/read, native remount persistence, k3s pod-recreation persistence, native cross-pool access denial and a controlled node-03 MON/OSD outage check passed on 2026-11-05; see [evidence](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md). See the [client evidence](../../../knowledge/okf/findings/2026-11-05-client-storage.md) for exact limits. Full VM reboot/outage, CephFS and later cases remain pending; the node-03 experiment stopped Ceph services only. Record date, exact versions, inputs, sanitized commands/output, expected versus observed behavior and recovery outcome in `vm/knowledge/okf/`.

| Case | Required evidence |
| --- | --- |
| Preflight | Three storage hosts inventoried; optional two clients inventoried for consumption exercises; three data disks positively identified; resource/network requirements satisfied |
| Baseline | Three MONs in quorum; MGR active/standby; three OSDs up/in; expected host placement; clean PGs and explained health state |
| Dashboard | HTTPS listener, administrator authentication, required password change, direct/provider URL access; see dashboard finding |
| Native RBD (extension) | Known checksums preserved after remount, service restart and client reboot; mount ordering prevents service writing into an unmounted local directory |
| k3s RBD | Bound PVC and correct CSI driver/pool; checksums survive pod deletion/recreation and client VM reboot |
| CephFS | Active/standby MDS; shared writes visible between native client and k3s; RWX works across two pods |
| OSD outage | Stop one OSD; record degraded state, continued eligible I/O and checksums; restore and wait for clean PGs |
| Storage-host outage | Stop one host including its MON; verify quorum, degraded I/O and checksums; restore to healthy baseline |
| Client outage | Restart client; verify remount/reattach, workload readiness and persisted data |
| Network interruption | In a controlled disposable environment, interrupt one selected endpoint without changing WireGuard configuration; record affected roles, observed I/O, recovery and management access |
| Negative credentials | Scoped client cannot administer the cluster or access unrelated pools/filesystems |
| Reproducibility | Repeat supported automation stages without accidental rebootstrap, duplicate resources or data loss |
| Teardown | Explicit target list, credential cleanup and documented retained/destroyed resources |

Run failures one at a time from a healthy baseline. Define timeout, stop conditions and recovery steps before each experiment. Never treat a snapshot in the same cluster as an independent backup. Performance results are illustrative for this shared VM/network lab and must include dataset size, client concurrency, replication settings and recovery state.

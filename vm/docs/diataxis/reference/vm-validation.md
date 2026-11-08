# VM validation matrix

Status on 2026-11-08: the Vagrant VMs are created and pass the preflight, see the [VM finding](../../../knowledge/okf/findings/2026-11-08-vagrant-vms.md). Every Ceph case below is still pending. Record date, exact versions, inputs, sanitized commands and output, expected versus observed behavior and recovery outcome in `vm/knowledge/okf/`.

| Case | Required evidence |
| --- | --- |
| Preflight | Three storage hosts checked; data and reserve disks identified by port and blank; host-only network reachable between VMs; resources as in the Vagrantfile |
| Baseline | Three MONs in quorum; MGR active/standby; three OSDs up/in; expected host placement; clean PGs and explained health state |
| Dashboard | Gateway HTTPS listener, administrator authentication, required password change, URL unchanged after MGR failover, opened from the host at https://192.168.56.12/ |
| Low space and expansion | Nearfull warning reached on the fill pool; three reserve OSDs added on `/dev/sdc` only; backfill completes; HEALTH_OK; memory per VM recorded |
| Monitoring | Prometheus, Alertmanager, Grafana and exporters running at expected counts, on pinned digests |
| Kernel client | Real kernel map attempted on kernel 7.0.0-43 before choosing NBD; retest on any new kernel or Ceph release |
| Native RBD (extension) | Known checksums preserved after remount, service restart and client reboot; mount ordering prevents service writing into an unmounted local directory |
| k3s RBD | Bound PVC and correct CSI driver/pool; checksums survive pod deletion/recreation and client VM reboot |
| CephFS | Active/standby MDS; shared writes visible between native client and k3s; RWX works across two pods |
| OSD outage | Stop one OSD; record degraded state, continued eligible I/O and checksums; restore and wait for clean PGs |
| Storage-host outage | Stop one VM with `vagrant halt`, including its MON, and sample cluster state during the outage, not only after it; verify quorum, degraded I/O and checksums; restore to healthy baseline |
| Client outage | Restart client; verify remount/reattach, workload readiness and persisted data |
| Network interruption | In a controlled disposable environment, bring down `enp0s8` on one VM; record affected roles, observed I/O, recovery and management access |
| Negative credentials | Scoped client cannot administer the cluster or access unrelated pools/filesystems |
| Reproducibility | `vagrant destroy` and `vagrant up`, then the manual steps or Ansible, give the same result; repeated Ansible runs cause no rebootstrap, duplicate resources or data loss |
| Teardown | `vagrant destroy` of the named VMs; credential cleanup; documented retained and destroyed resources |

Run failures one at a time from a healthy baseline. Define timeout, stop conditions and recovery steps before each experiment. Never treat a snapshot in the same cluster as an independent backup. Performance results are illustrative for VMs sharing one host computer and must include dataset size, client concurrency, replication settings and recovery state.

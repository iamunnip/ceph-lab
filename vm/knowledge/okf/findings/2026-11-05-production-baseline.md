# Production-aligned redeployment and failure tests, 2026-11-05

Status: deployed on five new playgrounds. All listed tests passed except kernel RBD, which is blocked. Open gaps are listed in [ADR 0010](../../../docs/adr/0010-production-aligned-baseline.md).

## Environment

Five new playgrounds, see [inventory](../../../inventory/iximiuz.json) and [versions](../../../config/versions.json). Ubuntu 26.04.1, kernel 6.1.167, 4 vCPUs, 9.8 GiB RAM, 20 GiB root disk, 80 GiB second disk. Cluster FSID `2e4e7220-d91f-11f1-bd47-0e9e435fac56`. Earlier findings describe the previous playgrounds, which no longer apply to these hosts.

## Network, measured

- `mesh0` MTU is 1280. From node-02, a don't-fragment ping with a 1172-byte payload passed and 1272 failed, as expected for this MTU.
- RTT from node-02: node-01 2.8 ms, node-04 3.6 ms, node-03 30 ms, node-05 29 ms.
- node-01 is the hub: it lists each peer separately, while the other nodes have one peer with allowed IPs `10.8.0.0/24`.
- WireGuard was not changed.

## Disks

The user approved wiping `/dev/vdb` on nodes 01 to 03, then asked for this to stand for future deployments, see [ADR 0009](../../../docs/adr/0009-standing-disk-wipe-approval.md). Each `/data` held only an empty `lost+found`. After the OSDs were created each disk showed `LVM2_member` with a `ceph_bluestore` volume. `/data` no longer exists on those hosts, and Ceph does not use any mount path for its data. On node-04 and node-05, `/dev/vdb` is still ext4 at `/data` and unused.

## Kernel RBD test

The `rbd` module exists and loads. Mapping a throwaway image with full features failed:

```text
rbd: failed to add secret 'client.native' to kernel
rbd: map failed: (22) Invalid argument
libceph: secret too big 32
```

Admin and client keys now start with `Ag`. [Ceph's release notes](https://docs.ceph.com/en/latest/releases/tentacle/) state that v20.2.4 introduced key type changes, alongside the CephX fix for CVE-2025-30156. Weakening the key was not tried, as repository rules forbid it. Which kernel version first accepts these keys was not established. The probe image was deleted.

## Deployment results

- HEALTH_OK, three MONs in quorum, three MGRs (one active, two standby), three OSDs up and in, 97 PGs active+clean.
- 240 GiB raw, about 76 GiB shared MAX AVAIL. Pools: `native-rbd` 32 PGs, quota 20 GiB. `k3s-rbd` 64 PGs, quota 40 GiB. Both size 3, min_size 2.
- Bootstrap left the known stray MON and MGR warning. A controlled `ceph mgr fail` cleared it.
- The gateway returned 200 for both TLS 1.2 and TLS 1.3, and dashboard login through it returned 201 with the password change still required. The earlier failure was caused by a TLS-1.3-only backend.
- `labctl expose port 6aec83720744dd9a83b9649f 443 --https` gave a private URL. Unauthenticated requests get a 302 to the iximiuz access page. **Not tested: an interactive browser session through this URL**, because it needs the user's iximiuz login.
- Native: `client.native` created and mapped `native-rbd/native-data`, and was denied listing `k3s-rbd` with `Operation not permitted`.
- k3s: Secrets encryption enabled, node Ready, CSI provisioner 7/7 and node plugin 3/3, 5 GiB PVC Bound. The marker checksum survived pod deletion. An on-demand etcd snapshot saved.
- Root disk use after deployment: 28% on nodes 01 and 02, 20% on node-03.

One command printed the start of the new dashboard user's bcrypt hash to the operator session. That password must be changed at first login, and the guide now discards this output.

## Failure tests

Run one at a time from a healthy state.

| Test | Result |
| --- | --- |
| MGR failover | Gateway URL unchanged. One 503 sample, then 200 for all later samples at 5-second intervals |
| node-04 reboot | Map, mount and nginx returned without help. Checksum `e046fa09...` unchanged |
| node-05 reboot | Node Ready, CSI and workload pods Running, NBD remapped. Checksum `9a2c0139...` unchanged |
| node-03 reboot | Back in 34 s, HEALTH_OK afterwards, disks and mesh intact, `/data` stays absent. Too fast to observe a degraded state, so this only proves boot recovery |
| node-03 Ceph services stopped for about 45 s | See below |
| node-01 hub reboot | See below |

### Sustained single-host outage

The first attempt was invalid: the status sampler ran on node-02, where cephadm was not installed, so the degraded state was never seen. This exposed a real gap, as node-02 had the admin keyring but no tool to use it. cephadm was installed there and the test repeated, sampling from node-02.

Observed while node-03 services were stopped: 2 of 3 MONs in quorum, 2 of 3 OSDs up, 104/312 objects degraded, all 97 PGs undersized and active. New writes with `sync` took 0.06 s on native and 0.22 s on k3s. Both original checksums were unchanged. HEALTH_OK returned 13 s after restoring services. This stops services only; it is not a power loss.

### Hub reboot

node-01 returned in 20 s. While it was down, a ping log on node-02 showed node-03 unreachable as well as node-01 (5 of 88 one-second samples), although neither node-02 nor node-03 failed. On node-04, repeated synced writes had one 27.38 s stall and no errors. The cluster returned to HEALTH_OK by itself, and both clients still wrote afterwards.

Quorum loss itself was not sampled, because the only sample taken ran on node-02 before cephadm was installed there. A longer hub outage was not tested. It cannot be held open without changing WireGuard or stopping the playground.

## Limitations

Not tested: power loss, a host absent long enough to be marked out, disk replacement, quota exhaustion, upgrades, backup and restore, CSI snapshots (k3s here has no VolumeSnapshot CRDs), performance under load. Whether reboots keep working across playground stop, start or expiry remains unknown.

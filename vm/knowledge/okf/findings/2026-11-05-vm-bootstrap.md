# VM cluster deployment, 2026-11-05

Status: cluster baseline deployed and smoke-tested.

## Observed environment

All five VMs were accessed using `labctl ssh <playground-id> -- <command>`. See [inventory](../../../inventory/iximiuz.json). Each has 4 vCPUs, 10024 MiB RAM, Ubuntu 26.04.1 userspace, kernel 6.1.167, a 20 GiB root disk and an approximately 80 GiB data disk. All five have an extra disk, including both clients, extending the initial supplied topology.

Existing mesh0 addresses are 10.8.0.1–5. All three storage addresses are reachable from nodes 02 and 03. Spoke-to-spoke ping was around 70 ms, hub-to-spoke around 35 ms. WireGuard was not changed. Identical eth0 addresses exist across separate playgrounds, so Ceph uses mesh0 addresses explicitly.

## Disk preparation

The user explicitly approved erasing `/dev/vdb` on nodes 01–03. Each was ext4 mounted at `/data` and contained only an empty `lost+found`. We checked mount source, serial, size and contents, backed up fstab to `/etc/fstab.before-ceph`, unmounted `/data`, removed its fstab entry, and removed the filesystem signature with `wipefs --all`. No root disk or client disk was modified.

The reported node-01 data device is 85899362304 bytes, 16 KiB larger than exactly 80 GiB. An initial equality check stopped safely before changes; the approved preparation used exact serials and a narrow byte-size range instead. Immediate `lsblk` filesystem output may briefly reflect cached metadata; `wipefs -n` is the signature check.

## Deployment choices

Pinned [versions](../../../config/versions.json). Cephadm 20.2.5 reports the expected Tentacle stable version. Upstream [release notes](https://www.ceph.io/en/news/blog/2026/v20-2-5-tentacle-released/) and [cephadm requirements](https://docs.ceph.com/en/tentacle/cephadm/install/) informed selection. Ubuntu 26.04/Podman 5.7 compatibility is being established by this lab's execution, not claimed as an upstream certification.

Installed Podman, LVM2, curl, Python, OpenSSH and Chrony from the host distribution. Chrony replaces systemd-timesyncd. Explicitly enabled Chrony and SSH because package service startup is suppressed by the image's policy-rc.d. Podman uses cgroup v2 and overlay. Node-01 `cephadm check-host` passed. No existing PVs/VGs were reported there.

Bootstrap uses `--mon-ip 10.8.0.1 --ssh-user laborant --skip-monitoring-stack --skip-dashboard`. Credentials remain on VMs. Dashboard and monitoring are deferred rather than exposing an unfinished management endpoint.

## SSH enrollment finding

The playground's existing `authorized_keys` did not end in a newline. Appending a public key without a leading newline concatenated it into the previous key's comment and caused authentication failure. Adding the Ceph public key as its own line fixed the fingerprint check. Preserve existing keys and always delimit appended keys with a leading and trailing newline. No SSH authentication algorithms or security policy were weakened.

OSD memory autotuning is disabled and the OSD memory target is set to 4 GiB to leave host headroom for MON/MGR and later MDS daemons. This is a target, not a hard memory limit. Logrotate was installed after bootstrap reported it missing.

## Verified baseline

- Cluster FSID: `3d7c41f0-d8e3-11f1-a04c-de08c7c5600d`.
- Final health: `HEALTH_OK`; 33 placement groups `active+clean`.
- MON quorum: node-01, node-02, node-03.
- MGR: node-02 active, node-01 standby following a controlled manager promotion.
- OSDs: osd.0 on node-01, osd.1 on node-02, osd.2 on node-03; all up/in.
- Both `.mgr` and `lab-smoke` pools use replicated size 3, min_size 2, CRUSH rule 0 with `chooseleaf` failure domain `host`.
- Reported raw storage: 240 GiB. Root filesystems remain ext4 and have about 16 GiB free per storage host. Data devices are now LVM-backed BlueStore OSDs, with no `/data` mount.
- All three storage hosts have synchronized Chrony clocks; final offsets were under 1 ms.
- From both node-04 and node-05, TCP connections to all three MONs on 3300 and advertised OSD v2 endpoints succeeded: 10.8.0.1:6802, 10.8.0.2:6800 and 10.8.0.3:6800. These daemon ports may change on redeployment. Earlier monitor reachability checks included legacy 6789.

The monitor service briefly reported node-03 unknown during enrollment convergence. Reapplying placement after the full host list was visible resolved the service error. OSDs briefly restarted as monitor maps changed and then converged without disk replacement.

Initial bootstrap MON/MGR stray warnings persisted despite managed daemon inventory. Promoting the existing standby using `ceph mgr fail node-01.tkqxrk` refreshed manager state and cleared the warnings; health warnings were not muted or disabled. This validates manager promotion only, not a full host failure/reboot experiment.

## Write/read test

Created the dedicated `lab-smoke` replicated pool, enabled its `rados` application, and explicitly set size 3/min_size 2. It was created with 8 PGs; autoscaling selected 32 PGs. Wrote object `hello` containing `Hello from the three-node Ceph lab.` plus a newline, read it back and verified both `cmp` and SHA-256:

```text
816728f48cab48c7a8aef337959fc170724069925d2f560c45f5fb4a4fa38328
```

The object and pool are retained for inspection. Temporary files were inside the disposable CLI container. This is a basic RADOS smoke test with admin credentials on the administration VM, not proof of scoped client permissions or application persistence.

The Python baseline validator passed all seven checks: health, monitor quorum, managers, OSD counts, host placement, replica policy and CRUSH host failure domains. These are historical results: the custom Python validator and shell wrapper were subsequently removed at the user's request. Current walkthroughs use the standard Ceph CLI directly.

## Outstanding validation

Native RBD, CSI, CephFS, full storage-host failure/reboot testing and playground data lifetime remain unvalidated. MTU behavior under larger transfers and complete recovery capacity/resource budgets remain open. At initial bootstrap, dashboard and monitoring were not deployed. Dashboard enablement is now recorded in the [follow-up finding](2026-11-05-dashboard.md); monitoring remains deferred. Node-02 has failed base-image units (`sshd-vsock.socket` and the removed `systemd-timesyncd.service`); Ceph services and Chrony run normally. These unrelated unit failures were recorded rather than altering playground access infrastructure.

# 2026-11-09: Stronger Ansible checks on new VMs

Question: do the checks added after the [end-to-end test](2026-11-09-end-to-end.md) work on a real cluster, and does the playbook still run with `changed=0` the second time? The new checks are: the pinned box version, the install-stage precheck, per-LV disk ownership, the full gateway certificate check, and the layout check by name and place.

## Environment

- Host: 12 CPU threads, 38 GiB RAM, 113 GB free disk, OpenSSL 4.1.0. Vagrant 2.4.9, VirtualBox 7.2, ansible-core 2.22.0.
- VMs: node-01 to node-03 from [vm/vagrant/podman/Vagrantfile](../../../vm/vagrant/podman/Vagrantfile), created new for this test. OpenSSL 3.5.5 on the VMs.
- Ceph 20.2.5 from [versions.yml](../../../configs/podman/versions.yml). Cluster fsid `bcd7adcc-dc0d-11f1-b502-0800271f81c1`.

## Procedure

1. `vagrant up` in `vm/vagrant/podman/`.
2. By hand on the new VMs: `sudo wipefs /dev/sdb /dev/sdc` and `sudo lvs --noheadings --separator '|' -o lv_name,devices,lv_tags` on node-01, `getent hosts node-01` on node-02.
3. `ansible-playbook site.yml` (first run).
4. By hand on the finished cluster: the fsid and `lvs` checks, the `openssl` check of all three gateways from the host, and `quorum_status`, `orch ps`, `osd dump` and `osd metadata` on node-01.
5. `ansible-playbook site.yml` (second run), then `ansible-playbook site.yml --tags install` alone.

## Observed

| Step | Observed | Matches the plan |
| --- | --- | --- |
| `vagrant up` | `Checking if box 'cloud-image/ubuntu-26.04' version '20260927.0.0' is up to date...`, `Machine booted and ready!` on all three | yes, the pinned version was used |
| `wipefs`, `lvs` before Ansible | no output from either | yes, blank disks |
| `getent hosts node-01` on node-02 before Ansible | no output, exit code 2 | yes, `/etc/hosts` is written by the setup stage |
| First run | `failed=0`; node-01 `changed=31`, node-02 `changed=5`, node-03 `changed=4`; `HEALTH_OK` | yes |
| Install-stage precheck, first run | ran on all three VMs: `getent hosts`, network, disks (all blank), `podman version 5.7.0`, cephadm version and `cephadm check-host` on node-01 and node-02 | yes |
| Gateway certificate, first run | cephadm's own certificate failed the check, so a new one was made; the wait for all gateways to serve it took 3 retries (about 1 minute); final check: `All 3 gateways serve one certificate (4D:10:62:...:8E:C0), CN=ceph-lab-gateway, valid for at least 30 more days, covering DNS:node-01, DNS:node-02, DNS:node-03, IP Address:192.168.56.11, IP Address:192.168.56.12, IP Address:192.168.56.13` | yes |
| Layout check, first run | `Monitors node-01, node-02, node-03; managers on node-01, node-02, node-03; OSDs up and in on node-01:sdb, node-02:sdb, node-03:sdb`; the manager wait passed on its first try | yes |
| `grep fsid /etc/ceph/ceph.conf` on node-01 | `fsid = bcd7adcc-dc0d-11f1-b502-0800271f81c1` (with a leading tab) | yes |
| `lvs ... lv_name,devices,lv_tags` on each VM | one line each: `osd-block-<uuid>\|/dev/sdb(0)\|ceph.block_device=/dev/ceph-<uuid>/osd-block-<same uuid>,...,ceph.cluster_fsid=bcd7adcc-...,ceph.osd_fsid=<uuid>,ceph.osd_id=0` (1 and 2 on node-02 and node-03)`,...,ceph.type=block,...` | yes |
| `openssl x509 -noout -subject -enddate -fingerprint -sha256 -ext subjectAltName -checkend 2592000` from the host | `subject=CN=ceph-lab-gateway`, `notAfter=Feb 11 07:35:45 2029 GMT`, the six SAN entries and `Certificate will not expire` on .11, .12 and .13, but `sha256 Fingerprint=34:AE:2F:...` | **no**: the fingerprint was wrong, see below |
| `quorum_status -f json-pretty` | `"quorum_names": ["node-01", "node-02", "node-03"]` | yes |
| `ceph orch ps`, `mgr.` lines | `mgr.node-01.xkfnbg`, `mgr.node-02.zhrfxu`, `mgr.node-03.oysohk`, all `running`, `REFRESHED 3m ago` | yes; the cached age confirms why the playbook uses `--refresh` |
| `ceph osd dump` | `osd.0 up   in`, `osd.1 up   in`, `osd.2 up   in` | yes |
| `ceph osd metadata -f json-pretty` | ids 0, 1, 2 with `"devices": "sdb"` and `"hostname"` node-01, node-02, node-03 | yes |
| Second run | `failed=0`, `changed=0` on node-01, node-02 and node-03; the certificate was kept (`Create a self-signed certificate ...` skipped) and reported as `4D:10:62:...:8E:C0` | yes |
| `--tags install` alone | `failed=0`, `changed=0`; the precheck ran on all three VMs, and the per-LV check passed `/dev/sdb` as an OSD of this cluster on each | yes |

## The fingerprint difference

The host's OpenSSL 4.1.0 printed `34:AE:2F:...` for the gateway certificate when `-ext subjectAltName` was in the same `openssl x509` command, and `4D:10:62:...` without it. The bytes served to the host and to node-01 were identical (`cmp`), and `openssl x509 -outform DER | sha256sum` gave `4d10623c...8ec0`, so `4D:10:62:...` is the real SHA-256 fingerprint. OpenSSL 3.5.5 on node-01 printed `4D:10:62:...` with the same options.

Change: the playbook ([gateway_cert_read.yml](../../../ansible/roles/install_podman/tasks/gateway_cert_read.yml)) and the [dashboard guide](../diataxis/how-to/dashboard.md) now read the fingerprint with a separate `openssl x509 -noout -fingerprint -sha256`. The second run above used this version.

## Result

All new checks passed on a real cluster without changing it, a second run gave `changed=0`, and a staged `--tags install` run checked its targets first. The manual commands gave the output the guides expect, except the fingerprint printed by OpenSSL 4.1.0 together with `-ext`, which is now read on its own.

## Limitations

- Only the passing path was seen on the VMs. The failing paths (a foreign OSD disk, an expiring or incomplete certificate, a wrong layout, a stale manager list) were tested on localhost with sample data, not on the VMs.
- The first-run case of an existing cluster built with the old layout template (one `changed` for the comment in the layout file) was not seen, because these VMs were new.
- Browser login and the first password change, the storage-host outage and recovery test, the reserve disk exercise and the client VMs are not part of this test.

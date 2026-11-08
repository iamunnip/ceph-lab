# 2026-11-08: Vagrant VMs created and checked

Question: do the VMs from the Vagrantfile meet the preflight for a manual cephadm install? Linked: Q001, Q003, Q006.

## Environment

- Host: 12 CPU threads, 38 GiB RAM, about 113 GB free disk.
- Vagrant 2.4.9, VirtualBox 7.2.22r175245, box `cloud-image/ubuntu-26.04` 20260927.0.0.
- [Vagrantfile](../../../vagrant/Vagrantfile) as committed with this finding. Only node-01 to node-03 were created; node-04 and node-05 are defined but not created.

## Procedure

1. `vagrant validate`, then `vagrant up` from `vm/vagrant/`.
2. The resources were changed from 4 CPU / 8 GB to 2 CPU / 6 GB during the run, then applied with `vagrant reload`.
3. The OS disk size was then lowered from 30 GB to 20 GB. VirtualBox cannot shrink a disk, so the three VMs, which held no data, were recreated with `vagrant destroy -f` and `vagrant up`. The checks below are from the recreated VMs.
4. In each VM: `nproc`, `free -m`, `df -h /`, `lsblk`, `/dev/disk/by-path`, `ip -4 -brief address`, `findmnt /`, `wipefs`, `pvs`, `timedatectl`, `sudo -n true`, ping between all three, `curl https://quay.io`, `apt-cache policy podman`.

## Expected and observed

| Check | Expected | Observed on node-01, node-02 and node-03 |
| --- | --- | --- |
| CPU and memory | 2 CPUs, 6 GB | 2 CPUs, 5920 MiB total |
| OS disk | 20 GB, root grown | `sda` 20G, `/dev/sda1` 19G, 2.0G used, 17G free |
| Data disk | 20 GB blank | `sdb` 20G, port `scsi-0:0:1:0`, no signature |
| Reserve disk | 20 GB blank | `sdc` 20G, port `scsi-0:0:2:0`, no signature |
| LVM | none | `pvs` empty |
| Disk serial | none expected | no serial, no `/dev/disk/by-id` entries |
| Host-only network | 192.168.56.11 to 13 | `enp0s8` with the planned address, MTU 1500 |
| NAT network | same on all | `enp0s3` 10.0.2.15 on every VM |
| Peer reachability | all reach all | all three reach each other; the host reaches node-02 |
| Internet | quay.io reachable | `HTTP/2 200` |
| Time | synchronized | chrony active; node-02 and node-03 `synchronized: yes`; node-01 still `no` about a minute after boot, while `chronyc tracking` showed `Leap status: Normal` and a 0.0006 s offset |
| sudo | passwordless | `sudo -n true` works for `vagrant` |
| OS and kernel | Ubuntu 26.04 | Ubuntu 26.04.1 LTS, kernel 7.0.0-34-generic |
| Podman | 5.7.0 available | candidate `5.7.0+ds2-3build1` after `apt-get update` |
| Hostname | unique | `node-01`, `node-02`, `node-03`; mapped to 127.0.1.1 in `/etc/hosts` |

The resize is set by `OS_DISK_SIZE` with `primary: true` in the Vagrantfile; it is not a Vagrant default. Vagrant grew the 10 GB box disk to 20 GB (`Disk 'node-0N-os' needs to be resized`), and cloud-init grew the root filesystem on first boot. 30 GB was first chosen for headroom but only 2.0 GB is used after boot, and 20 GB leaves enough for the Ceph and monitoring images and 2 GB of Prometheus data.

All three disks are now 20 GB, so size does not identify a disk; the root mount and the SCSI port do.

VirtualBox warned that the box's Guest Additions (6.0.0) do not match VirtualBox 7.2. Guest Additions only provide shared folders, clipboard and similar host integration. Shared folders are disabled, so the warning has no effect on the lab.

## Other evidence

The Vagrant 2.4.9 VirtualBox provider deletes disks that are removed from the Vagrantfile on the next reload: `cleanup_disks.rb` calls `close_medium`, which runs `VBoxManage closemedium <uuid> --delete`. This is why the reserve disk is attached from the start rather than added later.

## Operating system upgrade

First manual prerequisite step, run separately on node-01, node-02 and node-03 over `vagrant ssh`, as in the [deployment guide](../../../docs/diataxis/how-to/deploy-vm-cluster.md#update-and-upgrade-the-operating-system).

| Command | Observed on each node |
| --- | --- |
| `sudo apt-get update` | Fetched 32.5 MB, `Reading package lists...` |
| `apt list --upgradable` (node-01) | 76 packages, including kernel 7.0.0-34 to 7.0.0-43, systemd 259.5 to 259.9, OpenSSH 10.2p1-2ubuntu3.6 to 3.8 |
| `sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -y` | `72 upgraded, 0 newly installed, 0 to remove and 4 not upgraded`; kept back: `linux-headers-generic linux-headers-virtual linux-image-virtual linux-virtual`; needrestart deferred some service restarts |
| `sudo apt-get -s full-upgrade` (node-01) | 8 new `linux-...-7.0.0-43` packages, `0 to remove` |
| `sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y` | `4 upgraded, 8 newly installed, 0 to remove and 0 not upgraded`, grub regenerated |
| `vagrant reload node-01 node-02 node-03` (host) | `Machine booted and ready!` for each |
| `uname -r` | `7.0.0-43-generic` |
| `apt list --upgradable` | none |
| `ls /var/run/reboot-required` | file absent |

## Ceph prerequisites

Run separately on node-01, node-02 and node-03, as in [install the Ceph prerequisites](../../../docs/diataxis/how-to/deploy-vm-cluster.md#install-the-ceph-prerequisites). cephadm with Podman was chosen over Docker, a manual package install (Ubuntu has only 20.2.0; upstream 20.2.5 packages do not cover Ubuntu 26.04) and MicroCeph (snap 20.2.1).

| Command | Observed on each node |
| --- | --- |
| `sudo DEBIAN_FRONTEND=noninteractive apt-get install -y podman lvm2 curl python3 chrony logrotate` | lvm2, curl, python3, chrony and logrotate already newest; `0 upgraded, 31 newly installed, 0 to remove and 0 not upgraded` |
| `podman --version` | `podman version 5.7.0` |
| `systemctl is-active chrony ssh` | `active`, `active` |
| `sudo lvm version` | `LVM version: 2.03.31(2) (2025-02-27)` |
| `python3 --version` | `Python 3.14.4` |

## cephadm on node-01

Run on node-01 only, as in [install cephadm on node-01](../../../docs/diataxis/how-to/deploy-vm-cluster.md#install-cephadm-on-node-01).

| Command | Observed |
| --- | --- |
| `curl -fLsS https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm -o /tmp/cephadm-20.2.5` | exit 0, no output |
| `echo "4cc6...3f55  /tmp/cephadm-20.2.5" \| sha256sum -c` | `/tmp/cephadm-20.2.5: OK` |
| `sudo install -m 0755 /tmp/cephadm-20.2.5 /usr/local/sbin/cephadm` | exit 0, no output |
| `ls -l /usr/local/sbin/cephadm` | `-rwxr-xr-x 1 root root 1068332` |
| `sudo cephadm version` | `cephadm version 20.2.5 (ed27c686770872022f4c1de04f6b0ac5d5384ded) tentacle (stable)` |
| `sudo cephadm check-host` | Podman 5.7.0, systemctl, lvcreate present; chrony enabled and running; `Host looks OK` |

## Bootstrap on node-01

Run on node-01, as in [bootstrap the cluster](../../../docs/diataxis/how-to/deploy-vm-cluster.md#bootstrap-the-cluster). Started 13:38 UTC.

| Command | Observed |
| --- | --- |
| `sudo cephadm --image quay.io/ceph/ceph@sha256:4337ac8a...ef42c bootstrap --mon-ip 192.168.56.11 --ssh-user vagrant --skip-monitoring-stack --skip-dashboard` | exit 0, `Bootstrap complete.`; fsid `81c36e54-db79-11f1-9a8b-08002739f937`; Ceph 20.2.5 (ed27c686); `public_network` set to 192.168.56.0/24; no cluster network; MGR ports 9283 and 8765 checked free; logrotate.timer enabled |
| `sudo cephadm shell -- ceph -s` | `HEALTH_WARN`: 2 stray daemons, 1 stray host, `OSD count 0 < osd_pool_default_size 3`; mon quorum node-01; mgr `node-01.bsllkm` active |
| `sudo cephadm shell -- ceph orch ps` | `crash.node-01`, `mgr.node-01.bsllkm`, `mon.node-01` running on 20.2.5, image `26ed7271b5f8`; MGR 165M, MON 27.4M (limit 2048M) |
| `sudo cephadm shell -- ceph mon dump` | `[v2:192.168.56.11:3300/0,v1:192.168.56.11:6789/0] mon.node-01`; `auth_service_cipher aes256k` |
| `sudo cephadm shell -- ceph orch host ls` | `node-01 192.168.56.11 _admin` |
| `sudo cephadm shell -- ceph mon metadata node-01` | `"hostname": "node-01"`, addresses on 192.168.56.11 |
| `sudo cephadm shell -- ceph mgr fail node-01.bsllkm` | no output; `ceph mgr stat` epoch 10, available, same name |
| `sudo cephadm shell -- ceph health detail` (right after) | stray warnings still shown |
| `sudo cephadm shell -- ceph orch ps --refresh` and `ceph log last 20 debug cephadm` | refresh done; log shows only config updates, service discovery on 192.168.56.11:8765 and agent on 7150; no errors |
| `sudo cephadm shell -- ceph health detail` (about a minute later) | only `TOO_FEW_OSDS` |
| `free -m`, `df -h /`, `sudo podman images` | 841 MiB used, 5078 MiB available; root 4.7G used, 14G free; one image `quay.io/ceph/ceph` 1.52 GB |

Not certain whether the failover or the later refresh cleared the stray warning. Bootstrap deployed MON and MGR with default placement; the cluster spec replaces this with label placement.

## SSH key for cephadm

As in [let cephadm reach the other hosts](../../../docs/diataxis/how-to/deploy-vm-cluster.md#let-cephadm-reach-the-other-hosts).

| Command | Where | Observed |
| --- | --- | --- |
| `cat /etc/ceph/ceph.pub` | node-01 | one `ssh-ed25519` line ending `ceph-81c36e54-db79-11f1-9a8b-08002739f937` |
| `wc -l ~/.ssh/authorized_keys` | node-02, node-03 | `1` (the Vagrant key); `~/.ssh` already `700`, file already `600` |
| `mkdir -p ~/.ssh`, `chmod 700 ~/.ssh`, `touch ~/.ssh/authorized_keys`, `chmod 600 ~/.ssh/authorized_keys` | node-02, node-03 | each run separately, exit 0, no change |
| `printf '\n%s\n' '<ceph.pub line>' >> ~/.ssh/authorized_keys` | node-02, node-03 | exit 0 |
| `grep -c ssh- ~/.ssh/authorized_keys` | node-02, node-03 | `2`; `vagrant ssh` still works |
| `stat -c '%a %U %n' ~/.ssh ~/.ssh/authorized_keys` | node-02, node-03 | `700 vagrant`, `600 vagrant` |
| `sudo cephadm shell -- ceph cephadm check-host node-02 192.168.56.12` | node-01 | `node-02 (192.168.56.12) ok`, hostname matches, `Host looks OK` |
| `sudo cephadm shell -- ceph cephadm check-host node-03 192.168.56.13` | node-01 | `node-03 (192.168.56.13) ok`, hostname matches, `Host looks OK` |

## OSD memory target and cluster layout

As in [set the OSD memory target](../../../docs/diataxis/how-to/deploy-vm-cluster.md#set-the-osd-memory-target) and [apply the cluster layout](../../../docs/diataxis/how-to/deploy-vm-cluster.md#apply-the-cluster-layout).

| Command | Where | Observed |
| --- | --- | --- |
| `ceph config set osd osd_memory_target_autotune false` | node-01 | no output |
| `ceph config set osd osd_memory_target 1610612736` | node-01 | no output; `ceph config dump` shows both |
| `vagrant upload ../config/cluster-spec.example.yaml /home/vagrant/cluster-spec.yaml node-01` | host | uploaded; `cat` shows the expected spec |
| `lsblk`, `sudo wipefs /dev/sdb`, `readlink -f .../scsi-0:0:1:0` | each storage VM | `sdb` and `sdc` 20G blank; no signature; port 1 is `/dev/sdb` |
| `ceph orch device ls` | node-01 | node-01 `/dev/sdb` and `/dev/sdc` available |
| `ceph orch apply -i /spec.yaml` | node-01 | hosts node-01 to node-03 added with their 192.168.56.x addresses; mon, mgr and `osd.data-disks` scheduled |
| `ceph -s` right after | node-01 | still one MON, 0 OSDs |
| `ceph osd stat` about six minutes later | node-01 | `3 osds: 3 up, 3 in` |
| `ceph -s` | node-01 | `HEALTH_OK`; 3 MONs in quorum, leader node-01; MGR node-01.bsllkm active, standbys node-02.qbaqud and node-03.nlbica; 1 pool (`.mgr`), 1 PG active+clean; 82 MiB used of 60 GiB |
| `ceph osd tree` | node-01 | osd.0 node-01, osd.1 node-02, osd.2 node-03; class hdd; weight 0.01949 each |
| `ceph orch ps` | node-01 | crash, mgr, mon, osd on each host, 20.2.5; OSD MEM LIM 1536M, use 50 to 58 MiB; MGR 142 to 186 MiB; MON 26 to 45 MiB |
| `ceph orch ls` | node-01 | crash 3/3, mgr 3/3 label:mgr, mon 3/3 label:mon, osd.data-disks 3 label:osd |
| `ceph orch device ls` | node-01 | `/dev/sdb` not available on all three (`Has a FileSystem, Insufficient space (<10 extents) on vgs, LVM detected`); `/dev/sdc` available on all three |
| `lsblk`, `pvs`, `lvs`, `lvs -o lv_tags` | node-02 | `sdb` LVM2_member, VG `ceph-fe76d5fc-...`, LV `osd-block-092a14ae-...` 20G `ceph_bluestore`, PFree 0; tags `ceph.osd_id=1`, `ceph.cluster_fsid=81c36e54-...`, `ceph.type=block`, `ceph.encrypted=0`; `sdc` empty |
| `ceph mon dump` | node-01 | mons on 192.168.56.11, .12, .13, ports 3300 and 6789 |
| `ceph osd pool ls detail` | node-01 | `.mgr` replicated size 3 min_size 2 |
| `ceph df` | node-01 | 60 GiB raw, 82 MiB used |
| `free -m` | each storage VM | 818 to 904 MiB used, about 5000 MiB available |
| `df -h /` | node-02 | 4.7G used, 14G free |

Answers Q004 for MONs: all three bind to the host-only network.

## Second admin host, dashboard, monitoring and smoke test

| Command | Where | Observed |
| --- | --- | --- |
| curl, `sha256sum -c`, `install` of cephadm 20.2.5 | node-02 | `OK`, installed |
| `ls -l /etc/ceph` | node-02 | `ceph.client.admin.keyring` `-rw------- root`, `ceph.conf` |
| `sudo cephadm shell -- ceph health` | node-02 | `HEALTH_OK` |
| `ceph mgr services` | node-01 | `{}` |
| `ceph mgr module enable dashboard` | node-01 | no output |
| `ceph dashboard create-self-signed-cert` | node-01 | `Self-signed certificate created` |
| `install -m 0600 /dev/null /root/ceph-dashboard-password`, then `openssl rand ... \| sudo tee` | node-01 | file `-rw------- root`, 32 bytes; password not displayed |
| `ceph dashboard ac-user-create admin -i /pw administrator --pwd_update_required` | node-01 | output discarded; `ac-user-show` gives `["admin"]` |
| `ceph mgr services` | node-01 | `"dashboard": "https://192.168.56.11:8443/"` |
| five `ceph config set mgr mgr/cephadm/container_image_...` | node-01 | no output; `ceph config dump` shows all five digests |
| `vagrant upload ../config/monitoring-spec.yaml ...` then `ceph orch apply -i /spec.yaml` | host, node-01 | six services scheduled |
| `ceph orch ls` after about a minute | node-01 | exporters 3/3, prometheus 1/1; alertmanager, grafana, gateway 0/1 |
| `ceph orch ps` after about ten minutes | node-01 | alertmanager 0.28.1, grafana 12.3.1, prometheus 3.6.0 on node-02; node-exporter 1.9.1 and ceph-exporter on all three; `mgmt-gateway.node-02` `unknown` |
| `ceph health detail` | node-01 | `CEPHADM_FAILED_DAEMON: daemon mgmt-gateway.node-02 on node-02 is in unknown state` |
| `systemctl list-units --all 'ceph-*mgmt-gateway*'` | node-02 | unit `failed`; nothing on port 443 |
| `journalctl -u ceph-<fsid>@mgmt-gateway.node-02.service` | node-02 | `nginx: [emerg] host not found in upstream "node-01:8443" in /etc/nginx/nginx.conf:29` |
| `getent hosts node-01` | node-02 | no answer, exit 2; own name resolves to 127.0.1.1 |
| `printf '...' \| sudo tee -a /etc/hosts` with node-01 to node-05 | each storage VM | appended |
| `getent hosts` for the other nodes | each storage VM | 192.168.56.x answers |
| `ceph orch daemon restart mgmt-gateway.node-02` | node-01 | scheduled; unit `active`, nginx on 0.0.0.0:443 and [::]:443, local HTTPS 200 |
| `ceph health detail`, `ceph orch ls` | node-01 | `HEALTH_OK`; all services at expected counts |
| `curl -k https://192.168.56.12/` | host computer | `200`, title `Ceph` |
| `ceph osd pool create lab-smoke 8`, `application enable ... rados` | node-01 | created; `size 3 min_size 2 pg_num 8`; `9 pgs: 9 active+clean` |
| `rados put` and `rados get` via `cephadm shell --mount /home/vagrant/smoke:/smoke` | node-01 | read back `Hello from the three-node Ceph lab.`; both SHA-256 `816728f4...8328`; `cmp` exit 0 |
| `ceph osd map lab-smoke hello` | node-01 | `up ([0,2,1], p0) acting ([0,2,1], p0)` |

Root cause of the gateway failure: no name resolution between the VMs. Vagrant only maps a VM's own name, and cephadm writes the gateway's upstreams as `<hostname>:8443`. The MONs, MGRs and OSDs were unaffected because cephadm addresses them by IP. Fix recorded as a prerequisite in the deployment guide. Not yet checked whether the appended `/etc/hosts` lines survive a `vagrant reload`.

## Incident: dashboard wizard used the reserve disks

After the first login and password change (14:17), the dashboard showed a **Welcome to Ceph Dashboard** page saying "Add storage to begin using your cluster", although three OSDs existed. Its **Add storage** button was used at 14:19:24.

| Evidence | Observed |
| --- | --- |
| mgr log | `[POST] [201] [admin] /api/osd` from 192.168.56.1 at 14:19:24 |
| cephadm log | `Saving service osd.cost_capacity spec with placement *`, then `Deploying daemon osd.3 on node-01`, `osd.4 on node-03`, `osd.5 on node-02` |
| `ceph orch ls osd --export` | new spec `osd.cost_capacity`: `host_pattern: '*'`, `data_devices: rotational: 1` |
| `ceph osd metadata 3`, `4`, `5` | `devices: sdc` on node-01, node-03 and node-02 |
| `ceph health detail` | `BLUESTORE_SLOW_OP_ALERT: osd.5 observed slow operation indications in BlueStore`; `ceph osd perf` showed 0 ms on all six OSDs |
| `ceph df` | 120 GiB raw, 162 MiB used, 449 KiB data |

Recovery, chosen by the user ("restore the reserve"):

| Command | Observed |
| --- | --- |
| `ceph orch set-unmanaged osd.cost_capacity` | `Set unmanaged to True for service osd.cost_capacity`, so wiped disks are not claimed again |
| `ceph orch osd rm 3 4 5 --zap` | `Scheduled OSD(s) for removal.`; `osd rm status` showed 20, 13 and 18 PGs draining |
| `ceph orch osd rm status` until empty | done within a few minutes; `ceph osd tree` back to osd.0, osd.1, osd.2 |
| `lsblk /dev/sdc`, `sudo wipefs /dev/sdc` | each node: 20G, no filesystem, no signature |
| `ceph orch rm osd.cost_capacity` | `Removed service osd.cost_capacity` |
| `ceph orch device ls --refresh`, `ceph -s` | `/dev/sdc` available on all three; 3 OSDs; 60 GiB; 33 PGs active+clean; `HEALTH_OK`, slow op alert gone with osd.5 |

Lesson: the welcome page appears even when OSDs exist, and its wizard creates an OSD service that takes every spinning disk on every host. Documented as "do not click" in the dashboard guide.

## Incident: dashboard could not read Prometheus

The dashboard showed **Failed to execute prometheus: Error parsing Prometheus Alertmanager response: Expecting value**.

| Check | Observed |
| --- | --- |
| `ceph dashboard get-prometheus-api-host` and `get-alertmanager-api-host` | `https://node-02:29443/internal/prometheus` and `.../internal/alertmanager`; SSL verify `False` |
| `getent hosts node-02` inside the active mgr container | resolves; not a name problem |
| `curl -k https://node-02:29443/internal/alertmanager/api/v2/status` from node-01 | `400 No required SSL certificate was sent` (HTML) |
| gateway `nginx_internal_server.conf` | `ssl_verify_client on`, `ssl_client_certificate /etc/nginx/ssl/ca.crt` |
| dashboard `controllers/prometheus.py` | sends `mgr.get_localized_store("crt")`, the dashboard's own certificate, as client certificate |
| issuer of `mgr/dashboard/crt` | `O=IT, CN=ceph-dashboard`, self-signed by `ceph dashboard create-self-signed-cert` |
| issuer the gateway trusts | `CN=cephadm-root-81c36e54-db79-11f1-9a8b-08002739f937` |

Root cause: the guide's `create-self-signed-cert` step. Fix:

| Command | Observed |
| --- | --- |
| `ceph orch certmgr generate-certificates dashboard \| sudo tee /root/dashboard-cert/certs.json` into a 0600 file in a 0700 folder | keys `cert` and `key`; nothing printed |
| `jq -r .cert` and `.key` into 0600 files | done |
| `openssl x509 -noout -issuer -subject -ext subjectAltName` | `issuer=CN=cephadm-root-81c36e54-...`, `CN=192.168.56.11`, SAN `node-01`, `dashboard_servers`, `192.168.56.11` |
| `curl --cert ... --key ...` to both internal URLs | `401` instead of `400`: TLS accepted, Prometheus login is next, which the dashboard supplies |
| `ceph dashboard set-ssl-certificate -i`, `set-ssl-certificate-key -i` | `SSL certificate updated`, `SSL certificate key updated` |
| `ceph mgr fail node-01.bsllkm` | active manager now `node-02.qbaqud`; `https://192.168.56.12/` still 200 |
| `sudo rm -r /root/dashboard-cert` | removed |

## Incident: dashboard POST requests failed with 503 after the failover

After the failover to node-02.qbaqud, the browser got `503 Service Unavailable`, `"detail": "Keep on looking"`, request id `4e440636-b520-4962-a9df-ab8dabc9c348`.

| Check | Observed |
| --- | --- |
| gateway `nginx.conf` `upstream dashboard_servers` | `node-01:8443`, `node-02:8443`, `node-03:8443` |
| gateway `nginx_external_server.conf` `location /` | `proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;` (no `non_idempotent`) |
| `ss -lntp \| grep :8443` | `ceph-mgr` listens on all three hosts |
| node-03 mgr log, earlier | `Config not ready to serve, waiting: no certificate configured`, so standbys were not listening before a certificate existed |
| six `curl -X POST .../api/auth/check` from the host | `415 503 503 415 415 503` |
| three GET `/` | `200 200 200` |
| cephadm `get_dashboard_endpoints()` | lists every mgr daemon on the active port |
| upstream template in v20.2.5 and `main` | same rule, no `non_idempotent` |

Fix, recorded in [ADR 0011](../../docs/adr/0011-gateway-template-override.md):

| Command | Observed |
| --- | --- |
| `sudo cephadm shell -- cat /usr/share/ceph/mgr/cephadm/templates/services/mgmt-gateway/external_server.conf.j2 > /home/vagrant/external_server.conf.j2` | 197 lines; sha256 `fec000ed...a444`, equal to upstream v20.2.5 |
| `sed -i` adding `non_idempotent` to line 80 | `diff` shows only line 80 changed; saved in the repo as `config/mgmt-gateway/external_server.conf.j2`, sha256 `b75b3b92...0d2c` |
| `ceph config-key set mgr/cephadm/services/mgmt-gateway/external_server.conf -i /t.j2` | `set mgr/cephadm/services/mgmt-gateway/external_server.conf` |
| `ceph orch reconfig mgmt-gateway` | `Scheduled to reconfig mgmt-gateway.node-02 on host 'node-02'` |
| generated config on node-02 | line 40 ends in `http_504 non_idempotent;`; unit `active` |
| six POSTs and three GETs again | `415` six times, `200` three times |
| `ceph health detail` | `HEALTH_OK` |

Failover test with the override in place:

| Command | Observed |
| --- | --- |
| `ceph mgr fail node-02.qbaqud` | active manager now `node-03.nlbica` |
| six POSTs and three GETs right after | `502` for every request |
| `ceph mgr services`, then the same requests about 20 seconds later | dashboard `https://192.168.56.13:8443/`; GET `200` three times, POST `415` three times |

Reverted at the user's request:

| Command | Observed |
| --- | --- |
| `ceph config-key rm mgr/cephadm/services/mgmt-gateway/external_server.conf` | `key deleted`; `config-key exists` then reports `ENOENT` |
| `ceph orch reconfig mgmt-gateway` | scheduled; generated line 40 back to `... http_503 http_504;` without `non_idempotent`; unit `active` |
| `rm -f /home/vagrant/external_server.conf.j2` on node-01; lab copy removed from the repo | done |
| `ceph health` | `HEALTH_OK` |

The 503 issue for dashboard POST requests is open again; see [ADR 0011](../../docs/adr/0011-gateway-template-override.md).

## Prometheus stopped for now

At the user's request, after the override was reverted:

| Command | Where | Observed |
| --- | --- | --- |
| `ceph orch stop prometheus` | node-01 | `Scheduled to stop prometheus.node-02 on host 'node-02'` |
| `systemctl is-active ceph-<fsid>@prometheus.node-02.service` | node-02 | `inactive` |
| `ss -lntp \| grep :9095` | node-02 | nothing listening |
| `ceph orch ps --daemon-type prometheus --refresh` | node-01 | `prometheus.node-02 stopped` |
| `ceph health detail` | node-01 | `HEALTH_OK` |

The service spec, the pinned image and the stored metrics stay; `ceph orch start prometheus` brings it back. While stopped, alert rules are not evaluated, so Ceph alerts such as CephNodeDiskspaceWarning no longer appear in the dashboard, and its graphs and Grafana panels have no data. `ceph health` still reports cluster problems directly.

Started again at the user's request:

| Command | Where | Observed |
| --- | --- | --- |
| `ceph orch start prometheus` | node-01 | `Scheduled to start prometheus.node-02 on host 'node-02'` |
| `systemctl is-active`, `ss -lntp \| grep :9095`, `ceph orch ps --daemon-type prometheus --refresh` | node-02, node-01 | `active`; listening on 9095; `running`, 3.6.0 |

## Prometheus URL needs a login

The user reported the Prometheus URL as not working.

| Request from the host computer | Observed |
| --- | --- |
| `https://192.168.56.12/prometheus/` | `401 Unauthorized`, `Www-Authenticate: Basic` |
| `https://192.168.56.12/alertmanager/` | `401` |
| `https://192.168.56.12/grafana/` | `200` |
| `https://192.168.56.12:9095/` | no TLS connection (client certificate required) |
| `http://192.168.56.12:9095/` | `400` |

The gateway forwards `/prometheus` to Prometheus with its internal certificate; Prometheus then asks for HTTP basic authentication, which cephadm enables behind the mgmt-gateway. The credentials come from `ceph orch prometheus get-credentials`, run by the user in their own terminal so they stay out of the session. Whether the stored login is still a default value was not checked.

## Gateway on all three storage VMs

The user wanted the dashboard on `https://192.168.56.11/` too. Before: `.11:443` refused, `.11:8443` answered `503` (standby manager), `.12` worked.

| Command | Where | Observed |
| --- | --- | --- |
| `ceph orch host label add node-01 gateway`, same for node-03 | node-01 | `Added label gateway to host node-01`, `... node-03` |
| `monitoring-spec.yaml`: removed `count: 1` from `mgmt-gateway`; `cluster-spec.example.yaml`: `gateway` label on all three | repo | updated |
| `vagrant upload ../config/monitoring-spec.yaml ...`, `ceph orch apply -i /spec.yaml` | host, node-01 | six services scheduled |
| `ceph orch ps --daemon-type mgmt-gateway` about a minute later | node-01 | `mgmt-gateway.node-01`, `.node-02`, `.node-03` running on 443, nginx 1.26.1 |
| `ceph health detail` | node-01 | `HEALTH_OK` |
| GET `/` on `.11`, `.12`, `.13` from the host | host | `200` each; title `Ceph` |
| `/grafana/` and `/prometheus/` on `.11` | host | `200` and `401` (login prompt) |
| `ceph dashboard get-prometheus-api-host`, `get-alertmanager-api-host`, `GRAFANA_API_URL` | node-01 | now `https://node-01:29443/internal/prometheus`, `.../internal/alertmanager`, `https://node-01:443/grafana` |
| `openssl s_client -connect 192.168.56.11:443` | host | issuer `cephadm-root-81c36e54-...`; SAN `DNS:node-02, IP:192.168.56.12` only |

Each address works independently; there is no virtual IP. The POST 503 issue from [ADR 0011](../../docs/adr/0011-gateway-template-override.md) applies to every gateway.

## Follow-ups: password file, reboot, disk trend, gateway certificate

| Command | Where | Observed |
| --- | --- | --- |
| `sudo rm /root/ceph-dashboard-password` | node-01 | removed; `/root` empty. The user confirmed the dashboard loads on all three addresses and the Prometheus error is gone. |
| `ceph orch prometheus get-credentials` | node-01 | not run by the assistant (blocked by the session's safety check); left for the user |
| `ceph osd set noout` | node-01 | `noout is set` |
| `vagrant reload node-03` | host | `Setting hostname...`, `Machine booted and ready!` |
| `grep -n node- /etc/hosts`, `getent hosts node-01` | node-03 | the five `192.168.56.x node-0N` lines unchanged; resolves |
| gateway unit, `https://192.168.56.13/` | node-03, host | `active`; `200` |
| `ceph -s` | node-01 | 3 MONs in quorum, 3 OSDs up, 33 PGs active+clean; only warning `noout` |
| `ceph osd unset noout`, `ceph health detail` | node-01 | `noout is unset`; `HEALTH_OK` |
| `df -h /` | each storage VM | node-01 5.2G, node-02 6.3G (unchanged for about an hour), node-03 5.2G used; growth on node-01 and node-03 is the nginx image |
| `du -sh .../prometheus.node-02/data` | node-02 | 6.4M |
| cephadm `mgmt_gateway.py` | node-01 mgr | `mgmt_gw_cert` generated once for the first gateway host and reused |
| `openssl req -x509 ... subjectAltName=DNS:node-01..03,IP:192.168.56.11..13` into `/root/gw-cert` (0700) | node-01 | key 0600; valid until 2029-02-10 |
| `ceph orch certmgr cert set mgmt_gw_cert -i`, `key set mgmt_gw_key -i` | node-01 | `set correctly` for both |
| `ceph orch reconfig mgmt-gateway` | node-01 | new `nginx.crt` on disk, but the gateways still served the old certificate |
| `ceph orch restart mgmt-gateway` | node-01 | all three restarted; each address serves `CN=ceph-lab-gateway` with all six names; with that certificate trusted, curl verifies all three (`verify=0`, 200) |
| `sudo rm -r /root/gw-cert`, `ceph health detail`, `ceph orch ls mgmt-gateway` | node-01 | removed; `HEALTH_OK`; `3/3` |

Answers Q006 for reboots: added `/etc/hosts` lines, disks, addresses and Ceph daemons survive `vagrant reload`. `vagrant halt` and `up` remain untested.

## Alert: CephNodeDiskspaceWarning

The dashboard showed **CephNodeDiskspaceWarning**, "Host filesystem free space is getting low", active for 18 minutes. The rule is `predict_linear(node_filesystem_free_bytes{device=~"/.*"}[2d], 3600 * 24 * 5) ... < 0`: it fires when a straight line through up to 48 hours of free space reaches zero within 5 days. Prometheus had about an hour of data, all during image downloads. Actual use: root 26 % on node-01 and node-03, 35 % on node-02 (6.3G used, 12G free); `/boot` 17 %, `/boot/efi` 7 %. Treated as a false alarm from setup; expected to clear as the trend flattens. Not silenced. Recheck if still active after a day.

## Result

Core cluster, second admin host, dashboard, monitoring, HTTPS gateway and smoke test done. `HEALTH_OK`. The dashboard opens at `https://192.168.56.12/` from the host computer; first login and password change still to do by the user.

## Limitations

The client VMs node-04 and node-05 were not created. Stop and start persistence of disks and addresses across `vagrant halt` was not tested; one `vagrant reload` of the first VMs kept disks, addresses and hostnames.

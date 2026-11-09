# Deploy the three-VM Ceph cluster with Ansible

This guide runs the [manual deployment](deploy-vm-cluster.md) and the [dashboard setup](dashboard.md) as one Ansible playbook, with the install method `podman`. Every task in it comes from a command in those guides, and each task name says which command it runs. Read the manual guide first, so you know what each stage does.

Status: tested. On 2026-11-09, with ansible-core 2.22.0, the playbook built the cluster on new VMs (`failed=0`, `HEALTH_OK`), a second run reported `changed=0` on every VM, and a run on a cluster built by hand changed nothing in Ceph. See the [finding](../../findings/2026-11-09-end-to-end.md). The stronger checks (install-stage precheck, per-LV disk ownership, full gateway certificate check, layout by name) were tested the same day on new VMs: first run `failed=0`, second run and `--tags install` alone `changed=0`, see the [stronger-checks finding](../../findings/2026-11-09-stronger-checks.md). Its generated layout files match [cluster-spec.example.yaml](../../../../configs/podman/cluster-spec.example.yaml) and [monitoring-spec.yaml](../../../../configs/podman/monitoring-spec.yaml) apart from the header comment.

## What the playbook does

[site.yml](../../../../ansible/site.yml) first checks the install method (stage 0), then runs three stages. Each stage is one role:

| Stage | Role | Runs on | Does | Manual steps |
| --- | --- | --- | --- | --- |
| 0 | none | your computer | Check that `ceph_install_method` is `podman` or `snap`, load `vars/podman.yml` | |
| 1 | `prereq` | node-01 to node-03 | `apt-get update`, then install `podman lvm2 curl python3 chrony logrotate`. No OS upgrade | [Package lists](deploy-vm-cluster.md#refresh-the-package-lists), [prerequisites](deploy-vm-cluster.md#install-the-ceph-prerequisites) |
| 2 | `setup` | node-01 to node-03 | Add the lab hosts to `/etc/hosts`, check the host-only network, check the disks, check the Podman version, install the pinned cephadm on node-01 and node-02 | [Host names](deploy-vm-cluster.md#make-the-host-names-resolvable), [cephadm](deploy-vm-cluster.md#install-cephadm-on-node-01), [disk checks](deploy-vm-cluster.md#apply-the-cluster-layout) |
| 3 | `setup` (`precheck.yml`) | node-01 to node-03 | Read only: check the host names, the host-only network, the disks and the Podman version again, and the pinned cephadm and `cephadm check-host` on node-01 and node-02 | Same checks as stage 2 |
| 3 | `install_podman` | node-01 | Bootstrap, copy the cluster SSH key, set the OSD memory target, apply the layout, enable the dashboard, deploy monitoring and the gateway, check the gateway certificate, check `HEALTH_OK` and the layout by name | The rest of both guides |

Running it again is safe:

- Bootstrap runs only if `/etc/ceph/ceph.conf` does not exist yet.
- Each setting, certificate and user is checked first and changed only if needed.
- The layout files are applied again unchanged, which changes nothing in Ceph.
- It never upgrades the OS and never reboots a VM.

### Disk preparation

The playbook does not partition, format or wipe any disk. New Vagrant disks are blank, and `ceph-volume` creates its own LVM on the raw disk when it makes the OSD. The `setup` role only checks each disk in `ceph_data_devices` and `ceph_reserve_devices`:

- It must exist and sit on its SCSI port from `ceph_disk_by_path`.
- `wipefs` (read only, without `-a`) must show either nothing, or `LVM2_member` in a `ceph-...` volume group.
- An `LVM2_member` disk must also be an OSD of this cluster, checked on its own logical volumes: `lvs --separator '|' -o lv_name,devices,lv_tags` must show exactly one volume on that disk only, whose tags name that volume in `ceph.block_device=`, and hold `ceph.type=block`, an OSD id, an OSD fsid and exactly `ceph.cluster_fsid=` with the fsid from `grep fsid /etc/ceph/ceph.conf` on node-01. Before bootstrap there is no fsid, so any LVM disk stops the run.

Anything else stops the playbook before Ceph is installed. After the [low-space exercise](add-reserve-disks.md) has added `/dev/sdc`, set `ceph_reserve_devices_in_ceph: true` in `inventory/group_vars/ceph.yml`. Otherwise the final layout check stops the run, because it expects OSDs on `/dev/sdb` only. The install stage runs these disk checks again, so `--tags install` on its own also checks the disks as they are now. To start over, recreate the VMs with `vagrant destroy` and `vagrant up`.

## Where the settings are

| File | Holds |
| --- | --- |
| [inventory/hosts.yml](../../../../ansible/inventory/hosts.yml) | The VMs, their 192.168.56.x addresses, their `vagrant_machine` name (which picks each VM's SSH key, also for tasks that run on another VM with `delegate_to`), and the groups. A host gets a Ceph label by being in a group, for example `ceph_mon` for `mon`. |
| [inventory/group_vars/all.yml](../../../../ansible/inventory/group_vars/all.yml) | `ceph_install_method: podman`, login user, SSH key path, lab network and interface, and the pinned versions |
| [inventory/group_vars/ceph.yml](../../../../ansible/inventory/group_vars/ceph.yml) | Settings that are the same for every method: disks and waiting times |
| [vars/podman.yml](../../../../ansible/vars/podman.yml) | Settings of the podman method, one section per role: packages, cephadm, OSD memory, dashboard, monitoring and gateway certificate |

Pinned versions, checksums and image digests are read from [versions.yml](../../../../configs/podman/versions.yml), so the manual guide and the playbook use the same values. To use other VMs, change the addresses in `hosts.yml` and the user, key and interface in `all.yml`.

## Before you start

You need ansible-core on **your computer**. The playbook uses only modules built into ansible-core, so no extra collections are needed. The VMs must exist and be running. Create them with the [Vagrant guide](create-vagrant-vms.md); the preflight checks are part of the `setup` stage.

Run every command below on **your computer**, from the `ansible/` folder:

```sh
cd ansible
```

Check the Ansible version:

```sh
ansible --version
```

Expect `ansible [core 2.22.0]` or newer on the first line, and `config file = .../ansible/ansible.cfg`. If the config file line says `None`, you are in the wrong folder.

Show the groups and hosts:

```sh
ansible-inventory --graph
```

Expect `@ceph` with node-01 to node-03, `@ceph_bootstrap` with node-01 and `@ceph_admin` with node-01 and node-02.

Check that Ansible can log in to each storage VM. It uses the same key as `vagrant ssh`:

```sh
ansible ceph -m ansible.builtin.ping
```

Expect `node-01 | SUCCESS` with `"ping": "pong"`, and the same for node-02 and node-03. If you see `Permission denied` or `No such file`, the VMs are not created yet or the key path in `all.yml` is wrong. If you see `Connection timed out` right after `vagrant up`, the VMs are still starting; wait a minute and run the check again. On 2026-11-09 a playbook started straight after `vagrant up` timed out on all three VMs.

Check the playbook without running it:

```sh
ansible-playbook site.yml --syntax-check
```

Expect `playbook: site.yml`.

## Run the stages

You can run all three stages at once with `ansible-playbook site.yml`. Running one stage at a time shows you what each one does.

### Stage 1: prereq

```sh
ansible-playbook site.yml --tags prereq
```

Every stage first prints `Install method: podman`. Expect `changed` for the package install on a new VM, and a `PLAY RECAP` with `failed=0` for node-01 to node-03.

### Stage 2: setup

```sh
ansible-playbook site.yml --tags setup
```

Expect `failed=0` for all three VMs, and `podman version 5.7.0` from `Stop if Podman is not the tested version`. If that check fails, Ubuntu now ships another Podman version; test it and update `configs/podman/versions.yml`. The task `Show the check-host result` on node-01 and node-02 shows `Host looks OK`. If a disk check stops the run, the message names the disk and what it holds. Look at it by hand; the playbook never wipes it.

### Stage 3: install

```sh
ansible-playbook site.yml --tags install
```

The stage first runs the read-only checks of stage 2 again on node-01 to node-03: host names, network, disks, the Podman version, and cephadm with `cephadm check-host` on node-01 and node-02. If cephadm is missing, the message tells you to run `--tags setup` first. Only then does it bootstrap or create OSDs.

This takes about 15 minutes on a new cluster, mostly image downloads. Tasks that wait print `FAILED - RETRYING: ... (29 retries left)` while Ceph works in the background. That is Ansible checking again every 20 seconds, not an error: the task ends with `ok` once Ceph is ready. On 2026-11-09 the longest waits were:

| Task | Waited | Why |
| --- | --- | --- |
| `Wait until every monitoring service runs all its daemons` | about 4 minutes | node-02 downloads the Alertmanager and Grafana images |
| `Wait until every gateway serves the shared certificate` | about 2 minutes | the three gateways restart one after another |

A task fails only when it runs out of retries, after 10 minutes. Then it prints `fatal:` and the playbook stops.

At the end expect:

- `Stop if a gateway certificate is missing, expiring, wrongly named, incomplete or not shared` with `All 3 gateways serve one certificate`, its fingerprint, and all six names and addresses.
- `Wait for HEALTH_OK` with `ok`.
- `Stop if the cluster layout differs from the inventory` with `Monitors node-01, node-02, node-03; managers on node-01, node-02, node-03; OSDs up and in on node-01:sdb, node-02:sdb, node-03:sdb`.
- `Show the cluster status` with three MONs in quorum, three MGRs and `osd: 3 osds: 3 up, 3 in`.
- `Show how to open the dashboard` with the three gateway addresses.
- `PLAY RECAP` with `failed=0` for node-01.

## Check the result

Check with the same commands as the manual guide. Connect from the `vm/vagrant/podman/` folder:

```sh
vagrant ssh node-01
```

On **node-01**:

```sh
sudo cephadm shell -- ceph -s
```

Expect `HEALTH_OK`, three MONs, one active and two standby MGRs, and three OSDs up and in.

```sh
sudo cephadm shell -- ceph orch ls
```

Expect `mon 3/3`, `mgr 3/3`, `osd.data-disks 3`, `mgmt-gateway 3/3` and the monitoring services at 1/1 or 3/3.

```sh
sudo cephadm shell -- ceph orch device ls
```

Expect `/dev/sdb` as `Available No` and `/dev/sdc` as `Available Yes` on every host: the reserve disks were left alone.

Read the first dashboard password, privately:

```sh
sudo cat /root/ceph-dashboard-password
```

Open `https://192.168.56.11/`, `.12/` or `.13/` on your computer, sign in as `admin` and choose a new password. Then delete the file, as in the [dashboard guide](dashboard.md#create-your-login). The password never leaves node-01, and Ansible does not print it.

## Run it again

Run the full playbook a second time:

```sh
ansible-playbook site.yml
```

On a healthy cluster, expect `changed=0` for node-02 and node-03, and few or no changes on node-01. A change in the `prereq` stage only means new Ubuntu packages came out since the last run. A second run must never bootstrap again, add OSDs or change disks. Record the recap in the knowledge bundle.

## Recovery

If a run fails, read the failed task's message, fix the cause and run the same stage again. Every task checks the current state first, so you do not undo anything by hand. Do not delete `/etc/ceph/ceph.conf` to force a new bootstrap, and do not wipe a disk to get past a disk check. For Ceph problems, use the checks in the manual guide's [verification and recovery](deploy-vm-cluster.md#verification-and-recovery). To start completely over, recreate the VMs.

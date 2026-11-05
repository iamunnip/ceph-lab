# Deploy the three-VM Ceph cluster

This guide uses three Linux VMs on any provider or local hypervisor. Each has an OS disk and a second disk dedicated to Ceph. Additional client VMs are optional later extensions. You do not need iximiuz Labs or WireGuard. If you use iximiuz Labs, first read the [provider access guide](iximiuz-access.md).

Do not replay bootstrap or disk preparation on an existing cluster. The [deployment finding](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) describes our tested environment, rather than prerequisites specific to every reader.

## Choose your hosts and addresses

The commands below use this example. Replace every address, SSH username, hostname and disk path with your actual values before running them. Hostnames must match each machine's `hostname` output and be unique.

| Host | Example address | Purpose |
| --- | --- | --- |
| node-01 | 192.168.50.11 | Ceph and administration |
| node-02 | 192.168.50.12 | Ceph |
| node-03 | 192.168.50.13 | Ceph |

Provide one unused, disposable data disk per storage VM, around 80 GiB for this lab, in addition to the OS disk. Our tested baseline used 4 vCPUs and about 10 GiB RAM per VM; this is an example resource budget, not a certified minimum. Start with the [preflight](vm-preflight.md).

## Connect and prepare the hosts

From your computer, connect to node-01 using your own SSH user and address:

```sh
ssh ubuntu@192.168.50.11
```

Repeat the host preparation on node-02 and node-03. The examples use `ubuntu`, which must have passwordless sudo for cephadm. Use your VM image's existing sudo-enabled user or configure a dedicated account according to your environment's access policy. Verify `sudo -n true` succeeds. Cephadm must reach that account directly over SSH between storage hosts; a provider console alone is insufficient.

These installation commands are for Ubuntu/Debian-style hosts. Other Linux distributions need their corresponding package names and package manager; check the selected Ceph release's requirements before proceeding. All three hosts must already have routed connectivity to their actual Ceph addresses. Use a LAN, routed private network or existing VPN. Network provisioning is outside this guide. Permit SSH between storage hosts, MON TCP 3300/6789 as enabled, and the selected Ceph daemon range (normally TCP 6800–7568) between cluster hosts and, later, from any clients. Do not expose these ports broadly to the internet.

On nodes 01–03, install prerequisites and explicitly start services (the base image suppresses package service startup):

```sh
sudo apt-get update
sudo apt-get install -y podman lvm2 curl python3 openssh-server chrony logrotate
sudo systemctl enable --now chrony ssh
```

Record package versions. Confirm time synchronization and empty LVM state. Fetch the pinned standalone cephadm on node-01:

```sh
curl -fLsS https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm -o /tmp/cephadm-20.2.5
sudo install -m 0755 /tmp/cephadm-20.2.5 /usr/local/sbin/cephadm
sudo cephadm version
sudo cephadm check-host
```

Upstream requirements are documented in [Cephadm deployment](https://docs.ceph.com/en/tentacle/cephadm/install/). The observed Ubuntu/Podman combination is a lab result, not a claim of upstream certification.

## Data disks

On each storage VM, inspect disks before choosing a data device:

```sh
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
findmnt /
sudo pvs
sudo vgs
```

`lsblk` lists disks, filesystems and mounts. `findmnt /` identifies the root filesystem; also check its parent devices in `lsblk` if it uses partitions or LVM. Never select the OS disk. Size or an unmounted state alone does not prove a disk is disposable.

The commands later use `/dev/vdb` as an example only. If you use iximiuz Labs, follow its [disk steps](iximiuz-access.md#wipe-the-storage-data-disks). Your data disk may instead be `/dev/sdb` or an NVMe device, and each host may have a different path. Identify it by serial or stable device identity, verify ownership and confirm that all data may be destroyed. An eligible Ceph OSD device must have no mounted filesystem, filesystem signature, partitions or existing LVM ownership.

If it already contains data or is mounted, stop and back up or preserve it. Unmounting and removing signatures destroys access to the previous filesystem; do this only for a specifically identified, disposable disk. Update any corresponding persistent mount entry before reuse. This guide does not assume that `/data` exists. Our iximiuz-specific disk conversion is recorded in the [deployment finding](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md).

## Bootstrap and enrollment

On node-01, for a new cluster only:

```sh
sudo cephadm --image quay.io/ceph/ceph@sha256:4337ac8ac81d40df136e5eac92ca53bb52360a5e039f3d1f7935e729eb5ef42c bootstrap \
  --mon-ip 192.168.50.11 --ssh-user ubuntu \
  --skip-monitoring-stack --skip-dashboard
```

Bootstrap writes the cluster's public SSH key to `/etc/ceph/ceph.pub`. On node-01, display it:

```sh
cat /etc/ceph/ceph.pub
```

Copy the complete single line. Connect separately to node-02 and node-03 as the same account selected by `--ssh-user`. On each host, prepare the key file:

```sh
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Append the public key as a separate line, replacing the example text with the line you copied:

```sh
printf '\n%s\n' 'PASTE_THE_COMPLETE_CEPH_PUBLIC_KEY_HERE' >> ~/.ssh/authorized_keys
```

The leading newline handles files that lack a final newline. Preserve existing keys. Only the public key is copied; private keys and admin keyrings must stay out of Git.

## Apply the cluster layout

Instead of running one placement command per daemon, this guide describes the whole layout in one file and lets cephadm keep the cluster matching it. Ceph's documentation recommends this approach. See [cluster-spec.example.yaml](../../../config/cluster-spec.example.yaml).

Copy that file to node-01 as `~/cluster-spec.yaml`. Then edit it:

- Replace the three example addresses with yours.
- Replace `/dev/vdb` if your data disk has another path.

Host labels decide what runs where. `_admin` on two hosts means you can still administer the cluster if node-01 is lost. `monitoring` and `gateway` are used in a later step.

On node-01, set the memory target for each OSD. Our VMs have about 10 GiB of RAM and also run MON and MGR daemons, so automatic tuning is off and the target is 4 GiB. It is a target, not a hard limit:

```sh
sudo cephadm shell -- ceph config set osd osd_memory_target_autotune false
```

```sh
sudo cephadm shell -- ceph config set osd osd_memory_target 4294967296
```

Apply the layout. `--mount` makes your file visible inside the tools container. **This creates OSDs on the disk path named in the file, so confirm that path on every host first:**

```sh
sudo cephadm shell --mount $HOME/cluster-spec.yaml:/spec.yaml -- ceph orch apply -i /spec.yaml
```

Expect lines such as `Added host 'node-02'` and `Scheduled osd.data-disks update...`. Allow a few minutes for images to download. Watch progress:

```sh
sudo cephadm shell -- ceph -s
```

Expect three MONs, three MGRs (one active, two standbys) and three OSDs up and in. Bootstrap commonly leaves a "stray daemon" warning about node-01. If you see it, find the active manager's name, shown after `mgr:` in the `ceph -s` output:

```sh
sudo cephadm shell -- ceph mgr fail YOUR_ACTIVE_MANAGER_NAME
```

The warning should clear within a minute.

Ceph takes over the whole raw disk through LVM and BlueStore. It uses no filesystem or mount point, so never look for your data in a directory on the storage hosts. Confirm on each storage VM:

```sh
lsblk -o NAME,SIZE,TYPE,FSTYPE /dev/vdb
```

Expect `LVM2_member` on the disk and `ceph_bluestore` on the volume under it.

## Add a second administration host

The `_admin` label copies the configuration and admin keyring to node-02, but not the cephadm tool. Without it node-02 cannot run these commands. On **node-02**:

```sh
curl -fLsS https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm -o /tmp/cephadm-20.2.5
```

```sh
sudo install -m 0755 /tmp/cephadm-20.2.5 /usr/local/sbin/cephadm
```

```sh
sudo cephadm shell -- ceph health
```

Expect `HEALTH_OK`.

## Monitoring and HTTPS gateway

This step adds Prometheus, Alertmanager, Grafana, metric exporters and the HTTPS gateway. First [enable the dashboard](dashboard.md), since the gateway fronts it.

Pin each image by digest so a later redeploy cannot silently change versions. Run these on node-01. The values match [versions.json](../../../config/versions.json):

```sh
sudo cephadm shell -- ceph config set mgr mgr/cephadm/container_image_prometheus quay.io/prometheus/prometheus@sha256:76947e7ef22f8a698fc638f706685909be425dbe09bd7a2cd7aca849f79b5f64
```

```sh
sudo cephadm shell -- ceph config set mgr mgr/cephadm/container_image_alertmanager quay.io/prometheus/alertmanager@sha256:220da6995a919b9ee6e0d3da7ca5f09802f3088007af56be22160314d2485b54
```

```sh
sudo cephadm shell -- ceph config set mgr mgr/cephadm/container_image_grafana quay.io/ceph/grafana@sha256:2175aaa91c96733d86d31cf270d5310b278654b03f5718c59de12a865380a31f
```

```sh
sudo cephadm shell -- ceph config set mgr mgr/cephadm/container_image_node_exporter quay.io/prometheus/node-exporter@sha256:863b62ff9f392b6f472e22b4b670ad1cabb66b3042502ebd4b2cd192733e1bce
```

```sh
sudo cephadm shell -- ceph config set mgr mgr/cephadm/container_image_nginx quay.io/ceph/nginx@sha256:46755d957e58c5cf56362dd38cb218b023cd928abd3353006773bb7f204d656f
```

Copy [monitoring-spec.yaml](../../../config/monitoring-spec.yaml) to node-01 as `~/monitoring-spec.yaml`. It holds no addresses, so no edits are needed. Apply it:

```sh
sudo cephadm shell --mount $HOME/monitoring-spec.yaml:/spec.yaml -- ceph orch apply -i /spec.yaml
```

After a few minutes, list the services:

```sh
sudo cephadm shell -- ceph orch ls
```

Expect `alertmanager`, `grafana`, `prometheus` and `mgmt-gateway` at 1/1, and `node-exporter` and `ceph-exporter` at 3/3. Prometheus retention is capped at 2 GB because the OS disks in this example are small. Alertmanager has no notification receiver until you configure one, so alerts appear only in the dashboard.

## Verification and recovery

Connect to node-01:

```sh
ssh ubuntu@192.168.50.11
```

Check the cluster with these separate commands:

```sh
# Overall status: expect three MONs, one active and two standby MGRs, and three OSDs up/in.
sudo cephadm shell -- ceph -s

# Explain warnings: the finished baseline should say HEALTH_OK.
sudo cephadm shell -- ceph health detail

# Show one OSD under each storage host.
sudo cephadm shell -- ceph osd tree

# Show all managed daemons.
sudo cephadm shell -- ceph orch ps

# Check size 3 and min_size 2 on replicated pools.
sudo cephadm shell -- ceph osd pool ls detail
```

Inspect `ceph -s`, `ceph health detail`, `ceph orch ps`, `ceph osd tree`, `ceph osd pool ls detail` and `ceph osd crush rule dump` through the same remote CLI. Check clients can reach all advertised MON and OSD endpoints. Follow the deployment finding for measured validation and smoke-test results.

On failure, stop progression and inspect health, orchestrator events and daemon logs on the affected VM. Check `sudo cephadm ls` and systemd Ceph units. Do not rebootstrap, wipe an OSD or delete cluster data as a recovery shortcut. SSH authentication failures should first trigger key fingerprint/line-format checks and sudo checks. Disks already belonging to this cluster must not be wiped again.

Measured reboot and failure results are in the [production baseline finding](../../../knowledge/okf/findings/2026-11-05-production-baseline.md). Destructive teardown remains a separate stage.

## Try a small write/read test

A pool is a logical container for objects. This test uses RADOS, Ceph's underlying object storage layer. On node-01, create a dedicated test pool once:

```sh
sudo cephadm shell -- ceph osd pool create lab-smoke 8
sudo cephadm shell -- ceph osd pool application enable lab-smoke rados
sudo cephadm shell -- ceph osd pool set lab-smoke size 3
sudo cephadm shell -- ceph osd pool set lab-smoke min_size 2
```

Wait for `ceph -s` to show clean placement groups before testing. The pool autoscaler may adjust the initial count of 8 placement groups. Pool deletion is disabled by default as a safety measure, so this test pool stays unless you deliberately remove it. Our current iximiuz deployment skipped this pool and tested through the client volumes instead.

First connect to node-01, then enter the Ceph tools container:

```sh
ssh ubuntu@192.168.50.11
sudo cephadm shell
```

Run these commands inside that container. They create a small local file, upload it as an object, and download it again:

```sh
printf 'Hello from the three-node Ceph lab.\n' > /tmp/ceph-original.txt
rados -p lab-smoke put hello /tmp/ceph-original.txt
rados -p lab-smoke get hello /tmp/ceph-readback.txt
cat /tmp/ceph-readback.txt
```

The last command should print `Hello from the three-node Ceph lab.`. Compare the original with the downloaded copy:

```sh
sha256sum /tmp/ceph-original.txt /tmp/ceph-readback.txt
cmp /tmp/ceph-original.txt /tmp/ceph-readback.txt
```

Both checksums should match. `cmp` prints nothing and exits successfully when the contents are identical. This test overwrites only the named `hello` object in the dedicated test pool. It does not test application mounts or reboot persistence.

Leave the tools container when done:

```sh
exit
```

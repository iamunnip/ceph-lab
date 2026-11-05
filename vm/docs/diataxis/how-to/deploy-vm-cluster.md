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

The commands later use `/dev/vdb` as an example only. Your data disk may instead be `/dev/sdb` or an NVMe device, and each host may have a different path. Identify it by serial or stable device identity, verify ownership and confirm that all data may be destroyed. An eligible Ceph OSD device must have no mounted filesystem, filesystem signature, partitions or existing LVM ownership.

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

Each command below runs on node-01. `cephadm shell` supplies the Ceph tools from the container, and `--` separates those tools from cephadm options:

```sh
sudo cephadm shell -- ceph orch host add node-02 192.168.50.12
sudo cephadm shell -- ceph orch host add node-03 192.168.50.13
sudo cephadm shell -- ceph orch apply mon --placement=node-01,node-02,node-03
sudo cephadm shell -- ceph orch apply mgr --placement=node-01,node-02
sudo cephadm shell -- ceph config set osd osd_memory_target_autotune false
sudo cephadm shell -- ceph config set osd osd_memory_target 4294967296
sudo cephadm shell -- ceph orch device ls --wide --refresh
sudo cephadm shell -- ceph orch daemon add osd node-01:/dev/vdb
sudo cephadm shell -- ceph orch daemon add osd node-02:/dev/vdb
sudo cephadm shell -- ceph orch daemon add osd node-03:/dev/vdb
```

Confirm only the intended disks are eligible before adding OSDs. OSD memory target is 4 GiB, not a hard limit. Wait for image pulls and daemon convergence before diagnosing transient missing-daemon warnings.

Bootstrap above deliberately leaves dashboard creation to the next explicit step so credentials are set privately. The required UI step is the [dashboard guide](dashboard.md), which also covers enabling it on an existing cluster. Monitoring charts are deferred; the Ceph management UI remains part of this core setup.

## Verification and recovery

Connect to node-01:

```sh
ssh ubuntu@192.168.50.11
```

Check the cluster with these separate commands:

```sh
# Overall status: expect three MONs, an active and standby MGR, and three OSDs up/in.
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

Full host reboot/failure recovery, client integration and destructive teardown remain separate validation stages.

## Try a small write/read test

A pool is a logical container for objects. This test uses RADOS, Ceph's underlying object storage layer. On node-01, create a dedicated test pool once:

```sh
sudo cephadm shell -- ceph osd pool create lab-smoke 8
sudo cephadm shell -- ceph osd pool application enable lab-smoke rados
sudo cephadm shell -- ceph osd pool set lab-smoke size 3
sudo cephadm shell -- ceph osd pool set lab-smoke min_size 2
```

Wait for `ceph -s` to show clean placement groups before testing. The pool autoscaler may adjust the initial count of 8 placement groups. On our existing cluster this pool already exists; skip creation there.

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

# Deploy the three-VM Ceph cluster by hand

This guide installs Ceph with cephadm on the three Vagrant storage VMs. Every step is manual, so you see what each one does. The same steps will later become an Ansible playbook.

Before you start, [create the VMs](create-vagrant-vms.md) and pass the [preflight](vm-preflight.md). Do not replay bootstrap or OSD creation on a cluster that already exists.

Status: planned. These steps have not been run on the Vagrant lab yet. Record what you observe in the [knowledge register](../../../knowledge/okf/README.md).

## Lab values

| Host | Address | Purpose |
| --- | --- | --- |
| node-01 | 192.168.56.11 | Ceph, first administration host |
| node-02 | 192.168.56.12 | Ceph, second administration host, monitoring, dashboard gateway |
| node-03 | 192.168.56.13 | Ceph |

- SSH user: `vagrant`, with passwordless sudo.
- Ceph network: 192.168.56.0/24 on `enp0s8`. Never use the NAT address 10.0.2.15.
- Data disk: `/dev/sdb`, 20 GiB. This becomes the OSD.
- Reserve disk: `/dev/sdc`, 20 GiB. It stays out of Ceph until you [add it](add-reserve-disks.md).

If you run the VMs somewhere other than this Vagrantfile, replace the user, addresses and disk paths with yours.

## Prepare the hosts

Connect from the `vm/vagrant/` folder on your computer:

```sh
vagrant ssh node-01
```

### Update and upgrade the operating system

Do this on **node-01, node-02 and node-03** before installing anything, so every host starts from the same current packages and kernel.

Refresh the package lists:

```sh
sudo apt-get update
```

Expect `Reading package lists...` at the end. See what is waiting to be upgraded:

```sh
apt list --upgradable
```

On 2026-11-08 the box had 76 packages waiting, including a new kernel, systemd and OpenSSH. Upgrade them. `DEBIAN_FRONTEND=noninteractive` stops apt from waiting for an answer about changed configuration files, and keeps your current files:

```sh
sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -y
```

Expect a line like `72 upgraded, 0 newly installed, 0 to remove and 4 not upgraded`. The packages listed as `kept back` are the kernel: a new kernel comes as new packages, and `upgrade` never installs new packages.

Check what `full-upgrade` would do before running it. `-s` only simulates:

```sh
sudo apt-get -s full-upgrade
```

Expect only new `linux-...-7.0.0-43` packages (or a newer version) and `0 to remove`. If it wants to remove packages, stop and find out why. Install the new kernel:

```sh
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y
```

Expect `4 upgraded, 8 newly installed, 0 to remove and 0 not upgraded` and `Generating grub configuration file ... done`.

Leave the VM with `exit`. From the `vm/vagrant/` folder on **your computer**, reboot the three VMs so the new kernel and the upgraded systemd and SSH take effect. No Ceph runs yet, so this is safe:

```sh
vagrant reload node-01 node-02 node-03
```

Expect `Machine booted and ready!` for each. Connect again and check on **each storage VM**:

```sh
uname -r
```

Expect `7.0.0-43-generic` or newer, the same on all three.

```sh
apt list --upgradable
```

Expect only the `Listing...` line, with no packages under it.

### Install the Ceph prerequisites

On **node-01, node-02 and node-03**, install the container runtime and LVM tools. The package lists are already fresh from the upgrade step:

```sh
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y podman lvm2 curl python3 chrony logrotate
```

What each package is for:

| Package | Why cephadm needs it |
| --- | --- |
| `podman` | Every Ceph daemon (MON, MGR, OSD, monitoring) runs as a container from the pinned Ceph image, and so does `cephadm shell`. Podman is the runtime upstream recommends; it has no background daemon, so each container is a plain process under its own systemd unit. |
| `lvm2` | `ceph-volume` builds its LVM on the data disk when it creates an OSD. |
| `python3` | cephadm is a Python program. |
| `chrony` | Monitors need synchronized clocks. |
| `curl` | Downloads cephadm. |
| `logrotate` | Rotates the Ceph logs cephadm writes on the host. |

On 2026-11-08 the box already had all of them except Podman. Expect `lvm2 is already the newest version` and the same for curl, python3, chrony and logrotate, then `0 upgraded, 31 newly installed, 0 to remove and 0 not upgraded`. The 31 packages are Podman and its helpers, such as `crun`, `conmon` and `netavark`.

You do not create partitions, volume groups or logical volumes on `/dev/sdb` yourself. Leave the disk empty. When cephadm creates the OSD, its `ceph-volume` tool puts its own LVM volume group and logical volume on the disk and puts BlueStore on that. It needs the `lvm2` package for this. Ceph also has a raw mode without LVM, but LVM is the upstream default and this lab keeps it.

Check that time sync and SSH are running:

```sh
systemctl is-active chrony ssh
```

Expect `active` twice. Check the Podman version:

```sh
podman --version
```

Expect `podman version 5.7.0`, which matches [versions.json](../../../config/versions.json). Check LVM:

```sh
sudo lvm version
```

Expect `LVM version: 2.03.31(2)` on the first line.

### Make the host names resolvable

Ceph expects every host to resolve every other cluster host by name. The dashboard gateway, for example, sends requests to `node-01:8443`, and nginx refuses to start if that name does not resolve. Vagrant only maps each VM's own name, and there is no DNS for the lab names.

On 2026-11-08 this step was missing at first. The gateway then failed with `nginx: [emerg] host not found in upstream "node-01:8443"`, and the cluster showed `CEPHADM_FAILED_DAEMON`.

On **node-01, node-02 and node-03**, append the five lab hosts to `/etc/hosts`. `tee -a` appends and keeps the existing lines, and `> /dev/null` hides the echo:

```sh
printf '\n# ceph-lab hosts\n192.168.56.11 node-01\n192.168.56.12 node-02\n192.168.56.13 node-03\n192.168.56.14 node-04\n192.168.56.15 node-05\n' | sudo tee -a /etc/hosts > /dev/null
```

Run it only once per VM. Check from each VM that the other names resolve, for example on node-02:

```sh
getent hosts node-01
```

```sh
getent hosts node-03
```

Expect `192.168.56.11 node-01` and `192.168.56.13 node-03`.

## Install cephadm on node-01

cephadm is a single self-contained Python file. It bootstraps the cluster, deploys daemons on each host and opens `cephadm shell`. It is published under the `el9` path, but it is the same file for every Linux distribution.

On **node-01**, download the pinned standalone cephadm. `-f` makes curl fail on an HTTP error instead of saving an error page:

```sh
curl -fLsS https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm -o /tmp/cephadm-20.2.5
```

It prints nothing on success. Check that the file is the one this lab pinned:

```sh
echo "4cc603527d44ad8e753e1ed8a8e65cdcc05584fed74d1407425e0ae17ae13f55  /tmp/cephadm-20.2.5" | sha256sum -c
```

Expect `/tmp/cephadm-20.2.5: OK`. Stop if it fails. Install it as an executable owned by root:

```sh
sudo install -m 0755 /tmp/cephadm-20.2.5 /usr/local/sbin/cephadm
```

It prints nothing. Check the file:

```sh
ls -l /usr/local/sbin/cephadm
```

Expect `-rwxr-xr-x 1 root root 1068332`. Check the version:

```sh
sudo cephadm version
```

Expect `cephadm version 20.2.5 (ed27c686770872022f4c1de04f6b0ac5d5384ded) tentacle (stable)`. The commit matches `cephadm_commit` in [versions.json](../../../config/versions.json).

Check that the host meets cephadm's requirements:

```sh
sudo cephadm check-host
```

Expect:

```text
podman (/usr/bin/podman) version 5.7.0 is present
systemctl is present
lvcreate is present
Unit chrony.service is enabled and running
Host looks OK
```

If a line is missing, install or start what it names before bootstrapping. node-02 and node-03 do not need cephadm installed: when they are added to the cluster, the cephadm manager module copies its own copy to them over SSH. node-02 gets an installed cephadm later, only so it can act as a second administration host. Upstream requirements are in [Cephadm deployment](https://docs.ceph.com/en/tentacle/cephadm/install/). Ubuntu 26.04 with Podman 5.7.0 is this lab's combination, not an upstream certification.

## Bootstrap the cluster

On **node-01**, for a new cluster only:

```sh
sudo cephadm --image quay.io/ceph/ceph@sha256:4337ac8ac81d40df136e5eac92ca53bb52360a5e039f3d1f7935e729eb5ef42c bootstrap \
  --mon-ip 192.168.56.11 --ssh-user vagrant \
  --skip-monitoring-stack --skip-dashboard
```

- `--image` pins the exact Ceph 20.2.5 image by digest.
- `--mon-ip` puts the first monitor on the host-only network. Ceph derives the public network, 192.168.56.0/24, from it.
- `--ssh-user vagrant` is the account cephadm uses to manage the other hosts.
- The monitoring stack and dashboard are added in later steps.

Downloading the 1.52 GB image takes a few minutes. Bootstrap checks the host again, then reports each step. The important lines:

```text
Cluster fsid: <a new UUID>
Mon IP `192.168.56.11` is in CIDR network `192.168.56.0/24`
Internal network (--cluster-network) has not been provided, OSD replication will default to the public_network
Ceph version: ceph version 20.2.5 (ed27c686770872022f4c1de04f6b0ac5d5384ded) tentacle (stable)
Setting public_network to 192.168.56.0/24 in global config section
Wrote config to /etc/ceph/ceph.conf
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Wrote public SSH key to /etc/ceph/ceph.pub
Adding host node-01...
Bootstrap complete.
```

- The fsid identifies this cluster. Yours will differ.
- `public_network 192.168.56.0/24` confirms Ceph uses the host-only network, not NAT.
- The cluster-network line is expected: this lab has one network on purpose.
- `/etc/ceph/ceph.client.admin.keyring` is the full admin key. Never copy it into Git.
- Bootstrap also mentions `ceph telemetry on`. That sends anonymous usage reports to the Ceph project; this lab leaves it off.

With `--skip-dashboard`, no dashboard password is printed.

### Check the new cluster

On **node-01**, show the status:

```sh
sudo cephadm shell -- ceph -s
```

Expect one MON in quorum (`quorum node-01`), one active MGR such as `node-01.bsllkm`, and 0 OSDs. `HEALTH_WARN` is expected at this point, with up to three warnings:

| Warning | Meaning | What to do |
| --- | --- | --- |
| `OSD count 0 < osd_pool_default_size 3` | No OSDs exist yet | Nothing. It clears when the layout adds three OSDs. |
| `2 stray daemon(s) not managed by cephadm` | cephadm has not yet refreshed its list of the daemons bootstrap created | See below |
| `1 stray host(s) with 2 daemon(s) not managed by cephadm` | Same cause | See below |

List the daemons cephadm manages:

```sh
sudo cephadm shell -- ceph orch ps
```

Expect `mon.node-01`, `mgr.node-01.<suffix>` and `crash.node-01`, all `running` on version 20.2.5. The crash daemon collects crash reports.

Check that the monitor listens on the host-only address:

```sh
sudo cephadm shell -- ceph mon dump
```

Expect `[v2:192.168.56.11:3300/0,v1:192.168.56.11:6789/0] mon.node-01`. The `auth_service_cipher aes256k` line is the newer key format that old kernel clients cannot read.

### Clear the stray daemon warning

If the two stray warnings are there, first confirm that the host name and address agree on both sides:

```sh
sudo cephadm shell -- ceph orch host ls
```

```sh
sudo cephadm shell -- ceph mon metadata node-01
```

Expect host `node-01` with address `192.168.56.11` in the first output, and `"hostname": "node-01"` in the second. If they differ, stop and fix the hostname first.

If they agree, the warning is stale. Fail over the active manager, using the name shown after `mgr:` in `ceph -s`. With only one manager, this restarts it:

```sh
sudo cephadm shell -- ceph mgr fail node-01.bsllkm
```

It prints nothing. Make cephadm refresh its list of daemons now instead of waiting for its regular refresh:

```sh
sudo cephadm shell -- ceph orch ps --refresh
```

Check again:

```sh
sudo cephadm shell -- ceph health detail
```

Expect only `TOO_FEW_OSDS`. On 2026-11-08 the stray warnings were still shown right after the failover and had cleared about a minute later; the cephadm log showed no errors. To read that log:

```sh
sudo cephadm shell -- ceph log last 20 debug cephadm
```

## Let cephadm reach the other hosts

The cephadm manager module logs in to every other host over SSH, as `vagrant`, to deploy daemons there. Bootstrap created a key pair for this. The private key stays inside the cluster's configuration store; the public key is in `/etc/ceph/ceph.pub`. Each other host must trust that public key.

On **node-01**, display it:

```sh
cat /etc/ceph/ceph.pub
```

Expect one line that starts with `ssh-ed25519` and ends with `ceph-` followed by your cluster fsid. Copy the complete line.

This key pair was generated by bootstrap for the cluster. It is not your personal SSH key, and it is not the key `vagrant ssh` uses; those stay on your computer. The private half is stored in the cluster's configuration store as `mgr/cephadm/ssh_identity_key` and never leaves it. The file is only a copy of the public half. From any administration host you can also read it with:

```sh
sudo cephadm shell -- ceph cephadm get-pub-key
```

It prints the same line. Use this one when node-01 is not available.

Open a second terminal on your computer, go to `vm/vagrant/` and connect to node-02:

```sh
vagrant ssh node-02
```

Look at the existing key file first:

```sh
wc -l ~/.ssh/authorized_keys
```

Expect `1`: the key Vagrant uses for `vagrant ssh`. Keep it.

Make sure the folder and file exist with the permissions SSH requires. On the Vagrant box they already do, so these change nothing, but they make the step safe on other VMs. Create the folder if it is missing:

```sh
mkdir -p ~/.ssh
```

Only you may open it:

```sh
chmod 700 ~/.ssh
```

Create the file if it is missing:

```sh
touch ~/.ssh/authorized_keys
```

Only you may read or write it. SSH ignores the file if others can:

```sh
chmod 600 ~/.ssh/authorized_keys
```

Append the public key as a new line, replacing the example text with the line you copied:

```sh
printf '\n%s\n' 'PASTE_THE_COMPLETE_CEPH_PUBLIC_KEY_HERE' >> ~/.ssh/authorized_keys
```

`>>` appends, so the Vagrant key stays. The leading newline makes sure the new key starts on its own line even if the file has no final newline. Check:

```sh
grep -c ssh- ~/.ssh/authorized_keys
```

Expect `2`.

```sh
stat -c '%a %U %n' ~/.ssh ~/.ssh/authorized_keys
```

Expect `700 vagrant /home/vagrant/.ssh` and `600 vagrant /home/vagrant/.ssh/authorized_keys`.

Repeat on node-03.

Only the public key is copied. Private keys and admin keyrings must stay out of Git.

### Check that cephadm can log in

On **node-01**, let cephadm connect to each host with its own key and check it, exactly as it will when adding the host:

```sh
sudo cephadm shell -- ceph cephadm check-host node-02 192.168.56.12
```

Expect:

```text
node-02 (192.168.56.12) ok
podman (/usr/bin/podman) version 5.7.0 is present
systemctl is present
lvcreate is present
Unit chrony.service is enabled and running
Hostname "node-02" matches what is expected.
Host looks OK
```

```sh
sudo cephadm shell -- ceph cephadm check-host node-03 192.168.56.13
```

Expect the same for node-03. If it fails with a permission or authentication error, check the key line on that host: one line, complete, no line break inside it.

## Set the OSD memory target

On **node-01**. Each VM has 6 GB RAM and also runs a monitor and a manager, and node-02 also runs monitoring. Later each host gets a second OSD from the reserve disk. Turn off automatic tuning and give each OSD a 1.5 GiB target, so two OSDs use about 3 GiB:

```sh
sudo cephadm shell -- ceph config set osd osd_memory_target_autotune false
```

```sh
sudo cephadm shell -- ceph config set osd osd_memory_target 1610612736
```

Both print nothing. These are stored in the cluster's configuration database, so every OSD created later picks them up. Check:

```sh
sudo cephadm shell -- ceph config dump
```

Expect `osd basic osd_memory_target 1610612736` and `osd advanced osd_memory_target_autotune false` among the lines.

It is a target, not a hard limit. Check memory with `free -m` after adding the reserve disks.

## Apply the cluster layout

The whole layout is described in one file, and cephadm keeps the cluster matching it. Ceph's documentation recommends this over one command per daemon. See [cluster-spec.example.yaml](../../../config/cluster-spec.example.yaml). Its addresses already match the Vagrantfile.

Host labels decide what runs where. `_admin` on two hosts means you can still administer the cluster if node-01 is lost. `monitoring` and `gateway` are used in a later step.

The OSD part names only `/dev/sdb`. **Do not use `ceph orch apply osd --all-available-devices`**: it would also take the reserve disk `/dev/sdc`.

From the `vm/vagrant/` folder on **your computer**, copy the file to node-01:

```sh
vagrant upload ../config/cluster-spec.example.yaml /home/vagrant/cluster-spec.yaml node-01
```

Expect `Source: ../config/cluster-spec.example.yaml` and `Destination: /home/vagrant/cluster-spec.yaml`. On **node-01**, read it back:

```sh
cat /home/vagrant/cluster-spec.yaml
```

Expect three `host` entries with 192.168.56.11 to 13, the `mon` and `mgr` services placed by label, and the `osd` service with only `/dev/sdb` under `paths`.

This step creates OSDs, which takes over the disk. Confirm on **each storage VM** that the data disk is still blank:

```sh
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdb /dev/sdc
```

Expect `sdb` and `sdc`, 20G each, with no filesystem and no mount point.

```sh
sudo wipefs /dev/sdb
```

Expect no output.

```sh
readlink -f /dev/disk/by-path/pci-0000:00:0f.0-scsi-0:0:1:0
```

Expect `/dev/sdb`: the disk on SCSI port 1 is the data disk.

On **node-01**, list the disks cephadm can see:

```sh
sudo cephadm shell -- ceph orch device ls
```

Before the layout is applied, only node-01 is listed. Expect `/dev/sdb` and `/dev/sdc`, `hdd`, `20.0G`, `Available Yes`, and never `/dev/sda`.

Apply the layout. `--mount` makes your file visible inside the tools container. **This creates OSDs on `/dev/sdb` on every host labeled `osd`:**

```sh
sudo cephadm shell --mount /home/vagrant/cluster-spec.yaml:/spec.yaml -- ceph orch apply -i /spec.yaml
```

Expect:

```text
Added host 'node-01' with addr '192.168.56.11'
Added host 'node-02' with addr '192.168.56.12'
Added host 'node-03' with addr '192.168.56.13'
Scheduled mon update...
Scheduled mgr update...
Scheduled osd.data-disks update...
```

The command returns at once; the work happens in the background. node-02 and node-03 first download the 1.52 GB Ceph image, so allow several minutes. Watch progress:

```sh
sudo cephadm shell -- ceph -s
```

While it is still working you see fewer daemons and `OSD count 0 < osd_pool_default_size 3`. On 2026-11-08 all three OSDs were up about six minutes after applying. When it is done, expect:

```text
health: HEALTH_OK
mon: 3 daemons, quorum node-01,node-02,node-03
mgr: node-01.bsllkm(active), standbys: node-02.<suffix>, node-03.<suffix>
osd: 3 osds: 3 up, 3 in
pools:   1 pools, 1 pgs
usage:   82 MiB used, 60 GiB / 60 GiB avail
pgs:     1 active+clean
```

The one pool is `.mgr`, which the manager creates for its own data.

If a "stray daemon" warning appears here, handle it as in [clear the stray daemon warning](#clear-the-stray-daemon-warning).

Show one OSD under each host:

```sh
sudo cephadm shell -- ceph osd tree
```

Expect `osd.0` under node-01, `osd.1` under node-02 and `osd.2` under node-03, each `up`, class `hdd`, weight about `0.019` (the weight is the size in TiB).

Show every daemon cephadm manages:

```sh
sudo cephadm shell -- ceph orch ps
```

Expect `crash`, `mgr`, `mon` and `osd` on each of the three hosts, all `running` on 20.2.5. The `MEM LIM` of each OSD is `1536M`, the memory target set above.

Show the services and their placement:

```sh
sudo cephadm shell -- ceph orch ls
```

Expect `crash 3/3`, `mgr 3/3 label:mgr`, `mon 3/3 label:mon` and `osd.data-disks 3 label:osd`.

Check that the reserve disks were left alone:

```sh
sudo cephadm shell -- ceph orch device ls
```

Expect `/dev/sdb` as `Available No` on all three hosts, with the reasons `Has a FileSystem, Insufficient space (<10 extents) on vgs, LVM detected`: it is now an OSD. Expect `/dev/sdc` as `Available Yes` on all three hosts.

Ceph takes over the whole raw disk. `ceph-volume` created one LVM volume group and one logical volume on it, and BlueStore writes directly to that volume. There is no filesystem or mount point, so never look for your data in a directory. On each storage VM:

```sh
lsblk -o NAME,SIZE,TYPE,FSTYPE /dev/sdb /dev/sdc
```

Expect `LVM2_member` on `sdb`, a `ceph--<uuid>-osd--block--<uuid>` volume under it with `ceph_bluestore`, and nothing on `sdc`.

See the LVM that `ceph-volume` created:

```sh
sudo pvs
```

```sh
sudo lvs -o lv_name,vg_name,lv_size
```

Expect one physical volume `/dev/sdb` in a volume group `ceph-<uuid>` with no free space, and one logical volume `osd-block-<uuid>` of about 20 GiB. The OSD's identity is stored in tags on that volume:

```sh
sudo lvs --noheadings -o lv_tags
```

The output is one long comma-separated line. Look for `ceph.osd_id=` with this host's OSD number, `ceph.cluster_fsid=` with your fsid, `ceph.type=block` and `ceph.encrypted=0`. At boot, `ceph-volume` reads these tags to find and start the OSD.

Check the monitor addresses and the pool policy on **node-01**:

```sh
sudo cephadm shell -- ceph mon dump
```

Expect three monitors on 192.168.56.11, .12 and .13, each on ports 3300 and 6789.

```sh
sudo cephadm shell -- ceph osd pool ls detail
```

Expect `pool 1 '.mgr' replicated size 3 min_size 2`.

## Add a second administration host

The `_admin` label copies the configuration and admin keyring to node-02, but not the cephadm tool. On **node-02**:

```sh
curl -fLsS https://download.ceph.com/rpm-20.2.5/el9/noarch/cephadm -o /tmp/cephadm-20.2.5
```

```sh
echo "4cc603527d44ad8e753e1ed8a8e65cdcc05584fed74d1407425e0ae17ae13f55  /tmp/cephadm-20.2.5" | sha256sum -c
```

```sh
sudo install -m 0755 /tmp/cephadm-20.2.5 /usr/local/sbin/cephadm
```

Check that the label delivered the cluster files:

```sh
ls -l /etc/ceph
```

Expect `ceph.client.admin.keyring` readable only by root (`-rw-------`) and `ceph.conf`. The keyring is the full admin key, so treat node-02 as carefully as node-01.

```sh
sudo cephadm shell -- ceph health
```

Expect `HEALTH_OK`, now answered from node-02.

## Monitoring and HTTPS gateway

This step adds Prometheus, Alertmanager, Grafana, metric exporters and the HTTPS gateway. First [enable the dashboard](dashboard.md), because the gateway fronts it.

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

Each prints nothing. Check them:

```sh
sudo cephadm shell -- ceph config dump
```

Expect five `mgr/cephadm/container_image_...` lines with these digests, plus `container_image` with the Ceph image from bootstrap.

From `vm/vagrant/` on **your computer**, copy [monitoring-spec.yaml](../../../config/monitoring-spec.yaml) to node-01. It holds no addresses, so no edits are needed:

```sh
vagrant upload ../config/monitoring-spec.yaml /home/vagrant/monitoring-spec.yaml node-01
```

On **node-01**, apply it:

```sh
sudo cephadm shell --mount /home/vagrant/monitoring-spec.yaml:/spec.yaml -- ceph orch apply -i /spec.yaml
```

Expect six lines: `Scheduled node-exporter update...`, then the same for `ceph-exporter`, `prometheus`, `alertmanager`, `grafana` and `mgmt-gateway`.

The exporters and Prometheus start within about a minute. Alertmanager, Grafana and the gateway take longer, because node-02 downloads their images first. List the services:

```sh
sudo cephadm shell -- ceph orch ls
```

Expect `alertmanager`, `grafana` and `prometheus` at 1/1, and `mgmt-gateway`, `node-exporter` and `ceph-exporter` at 3/3. The gateway runs on every host labeled `gateway`, so the dashboard opens on all three storage VM addresses. Prometheus retention is capped at 2 GB to protect the 20 GB OS disks. Alertmanager has no notification receiver until you configure one, so alerts appear only in the dashboard.

### Stop or start Prometheus

To pause Prometheus without losing its configuration or stored metrics, on **node-01**:

```sh
sudo cephadm shell -- ceph orch stop prometheus
```

Expect `Scheduled to stop prometheus.node-02 on host 'node-02'`. Check:

```sh
sudo cephadm shell -- ceph orch ps --daemon-type prometheus --refresh
```

Expect `prometheus.node-02` with status `stopped`. `ceph health` stays `HEALTH_OK`. While it is stopped, the dashboard has no graphs, Grafana has no data and Ceph's alert rules are not evaluated; use `ceph health detail` for problems. Start it again with:

```sh
sudo cephadm shell -- ceph orch start prometheus
```

On 2026-11-08 the lab's Prometheus was stopped and started again this way. It was running again within seconds.

### Known issue: dashboard POST requests can fail with 503

The gateway sends each dashboard request to the three managers in turn. Standby managers answer `503 Keep on looking`, and the gateway is meant to try the next one. nginx does that for GET requests, but by default never retries a POST once sent. The dashboard uses POST for its login check and for actions, so in this lab about two in three of them can fail in the browser with `503 Service Unavailable`, `"detail": "Keep on looking"`. The cephadm template in Ceph 20.2.5, and in Ceph `main` on 2026-11-08, has this behavior.

A template override that adds nginx's `non_idempotent` flag fixed it in a test on 2026-11-08, but was reverted at the user's request and is not part of this lab. See [ADR 0011](../../adr/0011-gateway-template-override.md). Until there is a decision, if the dashboard shows that error, open the active manager directly. Find it on **node-01**:

```sh
sudo cephadm shell -- ceph mgr services
```

Open the `dashboard` address it prints, such as `https://192.168.56.13:8443/`. That address changes when another manager becomes active.

Right after a manager failover, the gateway briefly answers `502 Bad Gateway` to every request while the new active manager starts its dashboard. On 2026-11-08 the next check, about 20 seconds later, already worked.

## Verification and recovery

On **node-01**, check the cluster with these separate commands:

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

# Show raw and usable space: expect about 60 GiB raw.
sudo cephadm shell -- ceph df
```

Check that the monitors listen on the host-only address, not on 10.0.2.15:

```sh
sudo cephadm shell -- ceph mon dump
```

Expect `v2:192.168.56.11:3300` and the matching addresses for node-02 and node-03.

On failure, stop and inspect health, orchestrator events and daemon logs on the affected VM. Check `sudo cephadm ls` and the systemd Ceph units. Do not rebootstrap, wipe an OSD or delete cluster data as a recovery shortcut. For SSH failures, first check the key line format in `authorized_keys` and `sudo -n true` on the target host. Disks that already belong to this cluster must not be wiped again.

To start completely over, destroy the VMs from your computer with `vagrant destroy` and create them again. That deletes every disk, so there is nothing to wipe by hand.

## Try a small write and read test

A pool is a logical container for objects. This test uses RADOS, Ceph's underlying object storage layer. On node-01, create a test pool once:

```sh
sudo cephadm shell -- ceph osd pool create lab-smoke 8
```

```sh
sudo cephadm shell -- ceph osd pool application enable lab-smoke rados
```

Expect `pool 'lab-smoke' created` and `enabled application 'rados' on pool 'lab-smoke'`. Check the policy and placement groups:

```sh
sudo cephadm shell -- ceph osd pool ls detail
```

Expect `pool 2 'lab-smoke' replicated size 3 min_size 2 ... pg_num 8`. New pools get three copies by default.

```sh
sudo cephadm shell -- ceph pg stat
```

Expect `9 pgs: 9 active+clean`: 8 for this pool and 1 for `.mgr`. Pool deletion is disabled by default as a safety measure, so this pool stays unless you remove it on purpose.

The test uses a folder on node-01 that is mounted into the tools container, so every step is one command and the files stay on the host. On **node-01**, create the folder:

```sh
mkdir -p /home/vagrant/smoke
```

Write a small test file:

```sh
printf 'Hello from the three-node Ceph lab.\n' > /home/vagrant/smoke/ceph-original.txt
```

Upload it into the pool as an object named `hello`:

```sh
sudo cephadm shell --mount /home/vagrant/smoke:/smoke -- rados -p lab-smoke put hello /smoke/ceph-original.txt
```

It prints nothing on success. Download the object again into a new file:

```sh
sudo cephadm shell --mount /home/vagrant/smoke:/smoke -- rados -p lab-smoke get hello /smoke/ceph-readback.txt
```

Read it:

```sh
cat /home/vagrant/smoke/ceph-readback.txt
```

Expect `Hello from the three-node Ceph lab.`. Compare the two files:

```sh
sha256sum /home/vagrant/smoke/ceph-original.txt /home/vagrant/smoke/ceph-readback.txt
```

Expect the same checksum twice: `816728f48cab48c7a8aef337959fc170724069925d2f560c45f5fb4a4fa38328`.

```sh
cmp /home/vagrant/smoke/ceph-original.txt /home/vagrant/smoke/ceph-readback.txt
```

Expect no output, which means the files are identical. See where Ceph stored the object:

```sh
sudo cephadm shell -- ceph osd map lab-smoke hello
```

Expect a line ending like `up ([0,2,1], p0) acting ([0,2,1], p0)`: three copies on three OSDs, one per host. The first one, the primary, handles reads and writes. The order differs between objects.

## Next steps

- [Enable and open the dashboard](dashboard.md).
- [Fill the cluster and add the reserve disks](add-reserve-disks.md).
- Optional clients: [normal VM on node-04](native-rbd-client.md) and [k3s on node-05](k3s-rbd-client.md).

# Check the VMs before installing Ceph

These read-only checks confirm that the three storage VMs are ready for Ceph. They change nothing. Create the VMs first with the [Vagrant guide](create-vagrant-vms.md).

Run each check on **node-01, node-02 and node-03**. Connect from the `vm/vagrant/` folder on your computer, for example:

```sh
vagrant ssh node-01
```

## Operating system and resources

```sh
grep PRETTY_NAME /etc/os-release
```

Expect `Ubuntu 26.04.1 LTS`.

```sh
uname -r
```

The box ships with `7.0.0-34-generic`. The [deployment guide](deploy-vm-cluster.md#update-and-upgrade-the-operating-system) upgrades it, to `7.0.0-43-generic` on 2026-11-08. Note your value after the upgrade: the RBD client guides depend on it.

```sh
nproc
```

```sh
free -m
```

Expect 2 CPUs and about 5900 MiB total memory.

```sh
df -h /
```

Expect about 19G size and about 17G available.

## Hostname

```sh
hostname
```

Expect `node-01`, `node-02` or `node-03`. Ceph uses this name, so it must be unique.

Vagrant maps the hostname to `127.0.1.1` in `/etc/hosts`. That is fine for the VM's own name, because every Ceph command in these guides passes the 192.168.56.x address explicitly.

Each VM must also resolve the **other** VMs' names. The dashboard gateway, for example, refers to the managers as `node-01:8443`. Check:

```sh
getent hosts node-01
```

```sh
getent hosts node-02
```

```sh
getent hosts node-03
```

On a fresh Vagrant VM, only its own name answers (with `127.0.1.1`), and the others print nothing. That is expected here: the [deployment guide](deploy-vm-cluster.md#make-the-host-names-resolvable) adds the names. After that step, expect `192.168.56.11 node-01` and so on.

## Network

```sh
ip -4 -brief address
```

Expect `enp0s8` with the host's 192.168.56.x address. Ignore `enp0s3` (10.0.2.15): it is the NAT card and is the same on every VM.

Check that each storage VM reaches the other two. On node-01:

```sh
ping -c 2 192.168.56.12
```

```sh
ping -c 2 192.168.56.13
```

Expect `0% packet loss`. Run the matching checks from node-02 and node-03.

Check internet access for packages and container images:

```sh
curl -sI https://quay.io | head -1
```

Expect `HTTP/2 200`.

## Time

Ceph monitors need synchronized clocks:

```sh
timedatectl | grep synchronized
```

Expect `System clock synchronized: yes`. Right after boot it can still say `no` for a few minutes. Then check chrony directly:

```sh
chronyc tracking
```

Expect `Leap status     : Normal` and a `System time` offset well under a second. The box already runs chrony.

## Access

```sh
sudo -n true && echo sudo-ok
```

Expect `sudo-ok`. Cephadm logs in to the other storage VMs as `vagrant` and needs passwordless sudo there.

## Disks

List the disks:

```sh
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Expect:

| Disk | Size | Contents | Purpose |
| --- | --- | --- | --- |
| `/dev/sda` | 20G | root filesystem on `sda1`, plus boot partitions | OS, never use for Ceph |
| `/dev/sdb` | 20G | nothing | first OSD |
| `/dev/sdc` | 20G | nothing | reserve, for later |

Find the disk that holds the root filesystem:

```sh
findmnt -n -o SOURCE /
```

Expect `/dev/sda1`. That disk is never a Ceph target.

All three disks are 20G, so size never tells them apart. Confirm which is which by the controller port. Port 1 is the data disk and port 2 is the reserve disk:

```sh
ls -l /dev/disk/by-path/ | grep -v part
```

Expect `scsi-0:0:1:0 -> ../../sdb` and `scsi-0:0:2:0 -> ../../sdc`. The Vagrantfile attaches the data disk before the reserve disk, so the order is stable.

Confirm that neither extra disk has any signature, partition or LVM use:

```sh
sudo wipefs /dev/sdb /dev/sdc
```

Expect no output.

```sh
sudo pvs
```

Expect no output, or only lines that do not mention `sdb` or `sdc`.

New Vagrant disks are blank, so there is nothing to wipe. **If `sdb` or `sdc` shows a filesystem, partition or LVM signature, stop.** Find out where it came from before doing anything. Do not wipe it to make it fit.

## Go or no-go

Continue to the [deployment guide](deploy-vm-cluster.md) only when all three storage VMs pass every check above. Record anything unexpected in the [knowledge register](../../../knowledge/okf/README.md).

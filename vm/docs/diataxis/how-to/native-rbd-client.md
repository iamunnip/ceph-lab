# Use Ceph storage from a normal Linux VM

This exercise uses node-04 as a client of the three-host Ceph cluster. It creates a separate 10 GiB RBD image, mounts ext4 at `/srv/cephdata`, and serves a file from it with host nginx on port 18081. It does not use or erase node-04's local second disk.

The commands use the same example addresses as the deployment guide: storage nodes 192.168.50.11–13 and native client 192.168.50.14. Replace these with your own addresses. The observed iximiuz setup uses 10.8.0.1–4 and user laborant. See the [HA and capacity explanation](../explanation/ha-and-capacity.md).

## Create the pool and identity on node-01

Run each creation step only once. If the resources already exist, inspect them instead of recreating them.

```sh
sudo cephadm shell -- ceph osd pool create native-rbd
```

```sh
sudo cephadm shell -- ceph osd pool application enable native-rbd rbd
```

```sh
sudo cephadm shell -- rbd pool init native-rbd
```

Tell the autoscaler this pool should expect about a quarter of the cluster, so it allocates placement groups before data arrives:

```sh
sudo cephadm shell -- ceph osd pool set native-rbd target_size_ratio 0.25
```

Cap stored data at 20 GiB. A quota limits use, but it does not reserve space:

```sh
sudo cephadm shell -- ceph osd pool set-quota native-rbd max_bytes 21474836480
```

Confirm the pool uses three copies with a minimum of two, which are the defaults:

```sh
sudo cephadm shell -- ceph osd pool ls detail
```

Expect `replicated size 3 min_size 2` on the `native-rbd` line. Create an identity limited to this pool. Output is discarded so the key is not displayed:

```sh
sudo cephadm shell -- ceph auth get-or-create client.native mon 'profile rbd' osd 'profile rbd pool=native-rbd' mgr 'profile rbd pool=native-rbd' > /dev/null
```

On node-01, export only the scoped identity and non-secret cluster configuration into a private staging directory:

```sh
umask 077
mkdir -p ~/ceph-native-transfer
sudo cephadm shell -- ceph auth get client.native > ~/ceph-native-transfer/ceph.client.native.keyring
sudo cephadm shell -- ceph config generate-minimal-conf > ~/ceph-native-transfer/ceph.conf
scp ~/ceph-native-transfer/ceph.conf ~/ceph-native-transfer/ceph.client.native.keyring ubuntu@192.168.50.14:/tmp/
```

This assumes your operator SSH access to node-04 is configured. With iximiuz, run `labctl ssh` from your own computer and pipe the output of one playground into another; never paste credentials into Git or copy the admin keyring. On node-04, install the transferred files:

```sh
sudo install -d -m 0755 /etc/ceph
sudo install -m 0644 /tmp/ceph.conf /etc/ceph/ceph.conf
sudo install -m 0600 /tmp/ceph.client.native.keyring /etc/ceph/ceph.client.native.keyring
rm /tmp/ceph.client.native.keyring
```

Remove the private staged credential on node-01 after you have verified access.

## Install a compatible mapper on node-04

In production, the recommended way to map RBD is the kernel client. It does not work on our tested kernel, 6.1.167. A real map attempt failed with `libceph: secret too big 32`, because the kernel cannot load the 32-byte keys this Ceph release issues. The Ubuntu 20.2.0 `ceph-common` package cannot read these keys either. Weakening the keys is not an acceptable fix, as they are part of a security update.

If your kernel may be newer, try the kernel client first, on a throwaway image. If it maps, prefer it over the steps below. Otherwise this guide uses matching Ceph 20.2.5 tools and RBD-NBD, which runs the mapping in a userspace process. The container is privileged because it manages a host block device; nginx remains a normal host service.

```sh
sudo apt-get update
sudo apt-get install -y podman nbd-client nginx
sudo systemctl stop nginx
sudo modprobe nbd nbds_max=16
```

Load NBD on every boot with the same device limit:

```sh
printf 'options nbd nbds_max=16\n' | sudo tee /etc/modprobe.d/ceph-nbd.conf
```

```sh
printf 'nbd\n' | sudo tee /etc/modules-load.d/ceph-nbd.conf
```

Use the same pinned image as the cluster:

```sh
CEPH_IMAGE=quay.io/ceph/ceph@sha256:4337ac8ac81d40df136e5eac92ca53bb52360a5e039f3d1f7935e729eb5ef42c
sudo podman pull "$CEPH_IMAGE"
sudo podman run --rm --network host -v /etc/ceph:/etc/ceph:ro "$CEPH_IMAGE" rbd --id native create native-rbd/native-data --size 10G --image-feature layering,exclusive-lock,object-map,fast-diff,deep-flatten
```

The image is new Ceph storage. `exclusive-lock` is required by this exercise's exclusive NBD mapping, and the other features make snapshots and space reporting efficient. Do not map an ordinary ext4 image read-write on multiple VMs.

## Map and mount the new image

Copy the four files in [clients/native](../../../clients/native/) from your repository checkout to node-04. Install the map and mount units there:

```sh
sudo install -m 0644 ceph-native-map.service /etc/systemd/system/
sudo install -m 0644 srv-cephdata.mount /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ceph-native-map.service
sudo blockdev --getsize64 /dev/nbd0
```

The expected size is `10737418240` bytes (10 GiB). The example reserves `/dev/nbd0`; verify that it is unused before first mapping. The service waits for mapping before the mount and explicitly disconnects NBD during shutdown, avoiding a stale device connection after a container stops.

For this newly created, empty image only, check and create the filesystem:

```sh
sudo lsblk -f /dev/nbd0
sudo blkid /dev/nbd0
sudo mkfs.ext4 -m 0 /dev/nbd0
sudo mkdir -p /srv/cephdata
sudo systemctl enable --now srv-cephdata.mount
findmnt /srv/cephdata
df -h /srv/cephdata
```

**Do not run mkfs on an existing image containing data.** Check the mapping, size and absence of a filesystem before first formatting. The mounted filesystem will be a little smaller than 10 GiB because ext4 needs metadata.

## Run a service using the mount

On node-04, write the example page and install nginx configuration:

```sh
printf 'Persistent Ceph storage from node-04.\n' | sudo tee /srv/cephdata/index.html
sudo install -m 0644 nginx-ceph.conf /etc/nginx/conf.d/ceph-native.conf
sudo mkdir -p /etc/systemd/system/nginx.service.d
sudo install -m 0644 nginx-mount.conf /etc/systemd/system/nginx.service.d/ceph-mount.conf
sudo systemctl daemon-reload
sudo nginx -t
sudo systemctl enable --now nginx
curl http://127.0.0.1:18081/
```

Expect the page you just wrote. The nginx dependency prevents the service from starting without the Ceph filesystem. Open `http://YOUR_NODE04_ADDRESS:18081/` from a reachable host if desired. No public exposure is needed for the storage exercise.

## Verify persistence

On node-04, record the checksum, stop in service/mount/mapper order, and start again:

```sh
sudo sha256sum /srv/cephdata/index.html
sudo systemctl stop nginx
sudo systemctl stop srv-cephdata.mount
sudo systemctl stop ceph-native-map
sudo systemctl start srv-cephdata.mount
sudo systemctl start nginx
sudo sha256sum /srv/cephdata/index.html
curl http://127.0.0.1:18081/
```

Checksums should match. To test a full reboot, run `sudo systemctl reboot`, reconnect, and repeat the checksum and curl commands. The units start without help. Never reformat to solve a mapping problem; inspect systemd logs and NBD ownership first. The [client finding](../../../knowledge/okf/findings/2026-11-05-client-storage.md) records actual results.

## Return deleted space to Ceph

Deleting a file in ext4 does not free space in Ceph until the filesystem reports it. Enable the weekly trim on node-04:

```sh
sudo systemctl enable --now fstrim.timer
```

Run it once now to confirm it works:

```sh
sudo fstrim -v /srv/cephdata
```

Expect a line ending in `trimmed`.

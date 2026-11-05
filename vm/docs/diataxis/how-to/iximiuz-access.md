# Access the lab through iximiuz Labs

This optional guide adapts the [provider-independent deployment guide](deploy-vm-cluster.md) to iximiuz Labs. Readers using other VMs can skip it.

Our observed environment has five separate playgrounds already linked by WireGuard, but the core setup uses only node-01 to node-03. Nodes 04 and 05 now run the native and k3s client extensions; they remain unnecessary for the core cluster setup. See the [observed inventory](../../../inventory/iximiuz.json) for playground IDs and actual mesh addresses. Playground IDs change with every new set of VMs. Do not copy those IDs into your own environment; use your own running playgrounds. WireGuard setup is already complete in this environment and is not changed by these guides.

## Connect

From your computer, replace the ID below with your playground ID:

```sh
labctl ssh YOUR_PLAYGROUND_ID
```

If your playground contains multiple machines, select one explicitly:

```sh
labctl ssh YOUR_PLAYGROUND_ID --machine node-01
```

The current node-01 can be accessed with:

```sh
labctl ssh 6aec83710744dd9a83b96467
```

After connecting, the Ceph commands are the same as in the main guide. Use `laborant` instead of its example `ubuntu` account and the actual 10.8.0.1 to 10.8.0.3 mesh addresses instead of its example 192.168.50.11 to 192.168.50.13 addresses. The actual hostnames are node-01 to node-03.

## Provider-specific observations

- Different playgrounds have the same eth0 address; the existing mesh addresses are distinct. Ceph uses those mesh addresses.
- Additional disks were already ext4 filesystems mounted at `/data`. Nodes 01 to 03 are converted only under the [disk steps](#wipe-the-storage-data-disks) below. This is not a generic instruction to erase `/dev/vdb`.
- The existing authorized_keys file lacked a final newline. Always append a public key with a leading and trailing newline.
- Package installation did not start services automatically. Chrony and SSH were explicitly started.
- Playground reset/deletion durability remains unverified. Preserve important data outside the lab.

See the [finding](../../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) for evidence and known limitations.

## Wipe the storage data disks

New playgrounds mount an empty ext4 filesystem from `/dev/vdb` at `/data`. Ceph needs a blank disk, so this filesystem must go first. This applies only to iximiuz **node-01, node-02 and node-03**. Never run it on node-04 or node-05. Other providers follow the [main guide](deploy-vm-cluster.md#data-disks) instead.

The user has given [standing approval](../../adr/0009-standing-disk-wipe-approval.md) for this exact case, so an operator or agent does not need to ask again when every check passes. If any check gives a different result, stop and ask.

Run these on **each of node-01, node-02 and node-03**, one at a time. Confirm the host name, then look at the disk and its serial:

```sh
hostname
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,SERIAL
```

Expect `vda` mounted at `/`, and `vdb` of about 80G with type `ext4`, mounted at `/data`, with no child devices. Record the serial in your inventory.

Check that nothing is stored on the disk:

```sh
sudo find /data -mindepth 1
```

Expect only one line, `/data/lost+found`.

Check that no LVM or Ceph data exists. Install prerequisites from the [deployment guide](deploy-vm-cluster.md#connect-and-prepare-the-hosts) first, since they provide `pvs`:

```sh
sudo pvs
sudo ls -A /var/lib/ceph
```

Expect no physical volumes, and for the second command either no output or a "No such file or directory" error.

Keep a copy of the mount table, then unmount the filesystem and remove its line:

```sh
sudo cp /etc/fstab /etc/fstab.before-ceph
sudo umount /data
sudo sed -i '\|^/dev/vdb /data ext4|d' /etc/fstab
```

Remove the filesystem signature. This destroys the `/data` filesystem:

```sh
sudo wipefs --all /dev/vdb
```

Confirm the signature is gone:

```sh
sudo wipefs -n /dev/vdb
```

Expect a header line at most, with no `ext4` row.

## Check the cluster

After connecting to node-01, use the standard Ceph commands in the [main guide](deploy-vm-cluster.md#verification-and-recovery). No custom validator or inventory file is needed to run those commands.

## Expose the dashboard

First complete [dashboard enablement](dashboard.md) and the [gateway step](deploy-vm-cluster.md#monitoring-and-https-gateway). The gateway serves HTTPS on port 443 of the host labeled `gateway`, which is node-02 here. Exposure targets that playground, and it does not need to change after a manager failover.

On **your own computer**, replace the ID with your gateway host's playground ID:

```sh
labctl expose port YOUR_GATEWAY_PLAYGROUND_ID 443 --https
```

`--https` tells iximiuz that the service behind it uses HTTPS, including a self-signed certificate. Leave out `--public`, so visitors must sign in to your iximiuz account before they even reach the dashboard login. Expect a line with a URL ending in `.iximiuz.com`. Open it, sign in to iximiuz if asked, then sign in to the dashboard as `admin`.

Expose only this port, never MON or OSD ports.

If the gateway host is lost, the UI is unavailable until it returns or you move the `gateway` label to another host and expose that playground instead. Storage does not depend on the gateway.

An earlier deployment used a plain HTTP backend on port 18080, because the ingress could not negotiate the dashboard's TLS-1.3-only port 8443. The gateway accepts TLS 1.2 and 1.3, so that workaround is no longer needed. See [ADR 0010](../../adr/0010-production-aligned-baseline.md).

## Current deployment

| Item | Value |
| --- | --- |
| Gateway playground | node-02, `6aec83720744dd9a83b9649f` |
| Exposure command | `labctl expose port 6aec83720744dd9a83b9649f 443 --https` |
| [Dashboard URL](https://6aec83720744dd9a83b9649f-8373f8.node-eu-d241.iximiuz.com/) | Requires your iximiuz login, then the dashboard login |

Read the initial dashboard password privately from node-01:

```sh
labctl ssh 6aec83710744dd9a83b96467
```

```sh
sudo cat /root/ceph-dashboard-password
```

You must choose a new password at first login. Delete this file afterwards. These playgrounds expire about 24 hours after creation, and this deployment goes with them.

## Provider limits to remember

- **The WireGuard hub is node-01.** All mesh traffic between other nodes passes through it. Rebooting node-01 also cut node-02 off from node-03, and one client write stalled for 27 s. Ceph's replicas cannot protect against this. Other providers do not have this constraint unless they use a similar network.
- The mesh MTU is 1280, and RTT reaches about 30 ms. Do not read performance results from here as Ceph's capability.
- Guest reboots worked on all tested hosts. Playground stop, start and expiry remain untested.

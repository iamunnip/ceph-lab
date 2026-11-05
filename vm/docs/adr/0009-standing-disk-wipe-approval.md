# 0009: Standing approval to wipe iximiuz storage data disks

Status: accepted. Date: 2026-11-05.

## Context

Repository rules require explicit targets, ownership checks and confirmation before any destructive disk operation. Each new iximiuz playground set arrives with `/dev/vdb` as an empty ext4 filesystem mounted at `/data`, and new playgrounds have new disk serials. On 2026-11-05 the user approved wiping these disks on node-01, node-02 and node-03, then asked not to be asked again for this same case on future deployments.

## Decision

The user gives standing approval to wipe `/dev/vdb` on the iximiuz storage hosts node-01, node-02 and node-03 when preparing a new Ceph deployment, without asking again, provided every check below passes on that host immediately before the wipe:

1. The host is a playground listed by `labctl playground list` with the title node-01, node-02 or node-03, and `hostname` agrees.
2. `/dev/vdb` is not the root device or a parent of it.
3. `/dev/vdb` is either mounted only at `/data` as ext4, or unmounted with no signature at all.
4. `/data` contains nothing except an empty `lost+found`.
5. `/dev/vdb` has no partitions, is not an LVM physical volume and carries no Ceph signature.
6. No Ceph cluster exists on the host: `/var/lib/ceph` is absent or empty, and `sudo cephadm ls`, if cephadm is installed, lists no daemons.

The steps are the ones in the [iximiuz guide](../diataxis/how-to/iximiuz-access.md#wipe-the-storage-data-disks): back up `/etc/fstab`, unmount `/data`, remove its fstab line and run `wipefs --all /dev/vdb`. Record each serial in the [observed inventory](../../inventory/iximiuz.json).

## What this does not approve

Stop and ask the user if any check fails. This approval never covers:

- node-04 or node-05, including their `/dev/vdb` disks
- any root disk or any device other than `/dev/vdb`
- a disk with files, partitions, LVM or Ceph data, including an existing OSD
- zapping or re-creating OSDs on a running cluster
- any other provider, including VirtualBox and cloud VMs

Disk size or availability alone still never grants permission to wipe. Provider-independent guides keep their requirement that the reader identifies and confirms their own disk.

## Consequences

Redeploying on new playgrounds no longer pauses for a question that has one safe answer. The checks replace serial matching, because serials change with each playground and cannot be approved in advance. The user can withdraw this approval by changing this ADR.

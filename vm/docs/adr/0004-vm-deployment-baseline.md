# 0004: Pin and operate the VM deployment

Status: accepted; deployment planned. Date: 2026-11-05, revised 2026-11-08.

## Context

[ADR 0002](0002-vm-topology.md) selected the three-host cephadm topology. The Vagrant VMs report Ubuntu 26.04.1, kernel 7.0.0-34 from the box and 7.0.0-43 after upgrading, 2 vCPUs, 6 GB RAM, a 20 GB OS disk and two blank 20 GB disks on each storage VM. Every VM has the same NAT address, while the host-only addresses are unique. See the [VM finding](../../knowledge/okf/findings/2026-11-08-vagrant-vms.md).

## Decision

- Before installing anything, run `apt-get update`, `apt-get upgrade` and `apt-get full-upgrade` on every host, so the new kernel is installed too, then reboot. Check with a simulated `full-upgrade` that nothing is removed. All hosts then run the same packages and kernel. Packages are not pinned, so a later rebuild may get newer ones; record the kernel and Podman versions each time.
- Map storage hosts to node-01/02/03 at 192.168.56.11/12/13. node-04 is the native client and node-05 the k3s client. Use host-only addresses for all Ceph traffic; never the NAT address.
- Pin Ceph and standalone cephadm to 20.2.5, check the cephadm download against its SHA-256, and pin the Ceph container by digest. Use Podman 5.7.0 from Ubuntu. Record exact values in [versions.json](../../config/versions.json); this lab's use does not imply upstream certification of every host component.
- Keep observed machine facts in [inventory/vagrant.json](../../inventory/vagrant.json), separate from selected versions.
- Create the first OSDs only on `/dev/sdb`, named by path in the [cluster spec](../../config/cluster-spec.example.yaml). Leave the disk unpartitioned; `ceph-volume` creates its own LVM and BlueStore. Keep `/dev/sdc` as a reserve disk outside Ceph. Never use `--all-available-devices`.
- Disable OSD memory autotuning and set the per-OSD memory target to 1.5 GiB, so two OSDs per host fit in 6 GB next to MON, MGR and monitoring. This is not a hard limit.
- Keep a small `lab-smoke` RADOS pool for the write and read test. Use size 3/min_size 2 and host-level CRUSH placement.
- New Vagrant disks are blank, so no disk is wiped as part of a deployment. To start over, destroy and recreate the VMs.

## Consequences

The cluster depends on the host computer staying up. Recovery memory and future MDS placement must be measured before expanding scope. Credentials stay on the VMs, outside the repository.

See the [deployment guide](../diataxis/how-to/deploy-vm-cluster.md) for commands and expected output. This ADR refines ADR 0002 without superseding its topology decision. [ADR 0006](0006-direct-cli-validation.md) covers verification.

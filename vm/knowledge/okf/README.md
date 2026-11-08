# Open questions, known facts and findings

Updated: 2026-11-08. This register separates user-supplied facts, planned decisions, and observed evidence.

## Known facts

| ID | Fact | Source |
| --- | --- | --- |
| K001 | The lab runs on Vagrant and VirtualBox on the user's computer: 12 CPU threads and about 40 GB RAM | User, 2026-11-08 |
| K002 | Five VMs, each 2 CPUs and 6 GB RAM: node-01 to node-03 for Ceph, node-04 a normal VM using Ceph, node-05 for k3s | User, 2026-11-08; [Vagrantfile](../../vagrant/Vagrantfile) |
| K003 | Each storage VM starts with one 20 GB data disk; a second 20 GB disk is added to Ceph later to simulate running low on space | User, 2026-11-08 |
| K004 | Ceph is installed by hand first with plain manual commands only, one at a time, no scripts or loops; every command is recorded in the docs and later turned into Ansible | User, 2026-11-08 |
| K005 | VM work comes first; docs and knowledge must remain updated | User |
| K006 | Original repository code, configurations and documentation use the MIT License; third-party material retains its original terms | User-approved choice; [LICENSE](../../../LICENSE) |
| K007 | Use beginner-friendly documentation and direct Ceph commands; no custom validators or wrappers | User; [ADR 0006](../../docs/adr/0006-direct-cli-validation.md) |
| K008 | VM docs and knowledge belong under vm/; core setup needs three storage VMs, their data disks and the dashboard | User; [ADR 0007](../../docs/adr/0007-vm-owned-docs-and-dashboard.md) |
| K009 | The OSD data disk gets no manual partitions or LVM; `ceph-volume` creates its own LVM on the raw disk | User, 2026-11-08; [deployment guide](../../docs/diataxis/how-to/deploy-vm-cluster.md#prepare-the-hosts) |
| K010 | All three storage VMs pass the preflight: blank `sdb` and `sdc`, 20 GB OS disk, host-only network reachable | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| K011 | Vagrant deletes a disk removed from the Vagrantfile on the next reload | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| K012 | Vagrant VMs do not resolve each other's names; without `/etc/hosts` entries the mgmt-gateway fails with `host not found in upstream "node-01:8443"` | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| K013 | Core cluster, dashboard, monitoring and HTTPS gateway deployed by hand; `HEALTH_OK`; UI reachable at https://192.168.56.12/ from the host | [VM finding](findings/2026-11-08-vagrant-vms.md) |

## Open questions

| ID | Question / evidence needed | Resolution stage |
| --- | --- | --- |
| Q001 | Resolved: VM resources, OS, kernel and time sync recorded | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| Q002 | Is 6 GB RAM enough per storage VM with two OSDs at 1.5 GiB each, plus MON, MGR and, on node-02, monitoring? | Reserve disk exercise |
| Q003 | Resolved: data and reserve disks identified by SCSI port and confirmed blank | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| Q004 | Partial: all three MONs bind to 192.168.56.x; the gateway listens on 0.0.0.0:443, reachable only through the host-only network; OSD listeners still to check | Failure tests |
| Q005 | Resolved for the core: Ceph 20.2.5, cephadm and Podman 5.7.0 run all daemons on this box, once host names resolve | [VM finding](findings/2026-11-08-vagrant-vms.md) |
| Q006 | Partial: `vagrant reload node-03` kept disks, addresses, hostnames, the added `/etc/hosts` lines and all Ceph daemons; `vagrant halt` and `up` not tested | Failure tests |
| Q007 | Does kernel RBD on 7.0.0-43 accept the 32-byte keys of this Ceph release, so RBD-NBD is not needed? | Native client |
| Q008 | How long does backfill take after adding the reserve disks, and how much does node-02 memory rise? | Reserve disk exercise |
| Q009 | Which Ansible layout and inventory turns the manual steps into repeatable automation? | After manual deployment |

## Decisions and implementation status

Created and checked: node-01 to node-03 from the Vagrantfile, upgraded to current packages and kernel 7.0.0-43. Bootstrapped on node-01 with cephadm 20.2.5: fsid 81c36e54-db79-11f1-9a8b-08002739f937, public network 192.168.56.0/24. Deployed from the cluster spec: three MONs, three MGRs, three OSDs on `/dev/sdb`, `HEALTH_OK`, 60 GiB raw. Also deployed: node-02 as second admin host, dashboard with `admin` user, monitoring stack and HTTPS gateway on node-02, `lab-smoke` pool with a passed write and read test. Planned: the reserve disk exercise, a node-01 failover test, the two clients and Ansible. [ADR 0009](../../docs/adr/0009-vagrant-virtualbox-lab.md) records the platform choice and [ADR 0010](../../docs/adr/0010-production-aligned-baseline.md) the target configuration and its gaps.

## Findings

- [2026-11-08: Vagrant VMs created and checked](findings/2026-11-08-vagrant-vms.md).

Add a finding per experiment with: ID/date; linked question or hypothesis; exact versions/topology; procedure; expected and actual result; sanitized evidence; limitations; recovery result; and documentation/decision changes. Mark failed and inconclusive experiments honestly. Never store credentials in evidence.

## Local credential files

The repository ignores local inventories, environment files, cloud/Kubernetes credentials, Ceph keyrings, private-key/PEM files, runtime logs and Vagrant machine state. Example environment files remain trackable. These rules reduce accidental additions; review staged files for secrets before committing. Existing tracked files are not protected by ignore patterns.

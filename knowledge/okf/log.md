# Bundle update log

## 2026-11-09
* **Update**: Ansible waits now check every 20 seconds, 30 times (still 10 minutes), so `FAILED - RETRYING` lines halve. These lines mean Ansible is waiting, not failing.
* **Update**: Tested the [podman](methods/podman.md) method end to end on new VMs: by hand from the guides, then with [Ansible](tools/ansible.md). Both gave `HEALTH_OK` with 3 MONs, 3 MGRs and 3 OSDs. A second Ansible run reported `changed=0`. See `docs/podman/findings/2026-11-09-end-to-end.md`.
* **Update**: Fixed the Ansible SSH key under `delegate_to`: hosts now have `vagrant_machine` in the [inventory](ansible/inventory.md). Added SSH `timeout = 30` and `retries = 5` for VMs that have just booted.
* **Update**: The [install_podman](ansible/install-podman.md) monitoring wait and the manual guide use `ceph orch ls --refresh`; without it the list stayed at `0/N` after a manager restart.
* **Update**: Dropped the OS upgrade at the user's request. The manual guide and the [prereq](ansible/prereq.md) role run only `apt-get update` before the package install: no `upgrade`, no `full-upgrade`, no reboot. The VMs keep the box kernel 7.0.0-34.
* **Update**: Renamed `config/` to `configs/` at the user's request: `configs/podman/` holds the pinned versions, the cephadm layout files and the client files.
* **Update**: Removed `docs/podman/arc42/` at the user's request. Its facts are in this bundle; decisions stay in the ADRs.
* **Update**: Rebuilt this bundle in the kube-lab OKF layout: [index](index.md), this log and one folder per topic. The old register (`README.md`) is gone. Its facts are now pages, and each open question sits on the page it is about.
* **Update**: Moved the 2026-11-08 experiment record to `docs/podman/findings/2026-11-08-vagrant-vms.md`. The pages cite it as their source.
* **Creation**: Added the [snap](methods/snap.md) method next to [podman](methods/podman.md). Each has its own Vagrantfile in `vm/vagrant/<method>/`, and VirtualBox names start with `ceph-lab-podman-` or `ceph-lab-snap-`.
* **Creation**: Added the [Ansible](ansible/index.md) pages. One playbook in `ansible/` serves both methods through `ceph_install_method`. Syntax-checked with ansible-core 2.22.0, not yet run on the VMs.
* **Update**: Pinned versions moved from `vm/config/versions.json` to `configs/podman/versions.yml`. `vm/inventory/vagrant.json` was removed; it still listed the old 30 GB OS disk.
* **Update**: The VMs no longer exist (`vagrant status`: not created). The [cluster](ceph/cluster.md) facts are from 2026-11-08.

## 2026-11-08
* **Creation**: Created node-01 to node-03, upgraded them to kernel 7.0.0-43 and deployed Ceph 20.2.5 by hand with cephadm: 3 MONs, 3 MGRs, 3 OSDs, `HEALTH_OK`. See the [cluster](ceph/cluster.md).
* **Creation**: Added the [dashboard](ceph/dashboard.md), the [monitoring](ceph/monitoring.md) stack and a [gateway](ceph/gateway.md) on every storage VM.
* **Update**: Added [host name entries](networks/host-names.md) after the gateway failed with `host not found in upstream "node-01:8443"`.
* **Update**: Lowered the [OS disk](storage/os-disk.md) from 30 GB to 20 GB and recreated the VMs.
* **Update**: Removed three OSDs that the dashboard's **Add storage** wizard had created on the [reserve disks](storage/reserve-disk.md).

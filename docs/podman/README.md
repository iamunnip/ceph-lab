# Podman track

This track installs Ceph with cephadm, which runs every Ceph daemon as a Podman container. The [snap track](../snap/README.md) uses the same VMs with another install method; run only one lab at a time.

Set up a Ceph cluster on three VirtualBox VMs created by Vagrant, use each VM's data disk for storage, and enable the Ceph Dashboard. Two more VMs use the cluster's storage. Ceph is installed by hand first, then with the [Ansible playbook](../../ansible/README.md) built from the same commands, with `ceph_install_method: podman`.

## Start here

1. [Create the VMs with Vagrant](diataxis/how-to/create-vagrant-vms.md).
2. [Check the VMs](diataxis/how-to/vm-preflight.md).
3. [Deploy Ceph by hand](diataxis/how-to/deploy-vm-cluster.md).
4. [Enable and open the dashboard](diataxis/how-to/dashboard.md) at `https://192.168.56.11/`, `.12/` or `.13/`.
5. [Fill the cluster and add the reserve disks](diataxis/how-to/add-reserve-disks.md).

After the manual install, [deploy with Ansible](diataxis/how-to/deploy-with-ansible.md) runs steps 3 and 4 as one playbook. Both were tested end to end on new VMs on 2026-11-09, see the [finding](findings/2026-11-09-end-to-end.md).

## Machines

Every VM has 2 CPUs, 6 GB RAM and a 20 GB OS disk. All five together need 10 CPUs and 30 GB RAM.

| Host | Address | Purpose | Extra disks |
| --- | --- | --- | --- |
| node-01 | 192.168.56.11 | MON, MGR, OSD and administration | 20 GB data `/dev/sdb`, 20 GB reserve `/dev/sdc` |
| node-02 | 192.168.56.12 | MON, MGR, OSD, second administration host, monitoring and HTTPS gateway | same |
| node-03 | 192.168.56.13 | MON, MGR and OSD | same |
| node-04 | 192.168.56.14 | Normal VM using a Ceph block device (optional) | none |
| node-05 | 192.168.56.15 | Single-node k3s using Ceph through CSI (optional) | none |

`vagrant up` starts node-01 to node-03. Start the clients by name when you need them.

Both extra disks are for Ceph data. The data disk is the first OSD. The reserve disk stays empty until the cluster runs low on space, then becomes a second OSD on each host. The cluster uses three replicas across hosts, with a minimum of two for I/O: 60 GiB raw gives about 20 GiB usable at first, and 120 GiB raw about 40 GiB after adding the reserve disks.

## Consume storage and understand HA

- [Capacity and Ceph HA](diataxis/explanation/ha-and-capacity.md).
- [Normal VM on node-04](diataxis/how-to/native-rbd-client.md): 5 GiB RBD volume mounted at /srv/cephdata, served by host nginx.
- [k3s on node-05](diataxis/how-to/k3s-rbd-client.md): Ceph-CSI, an explicit RBD StorageClass, a 5 GiB PVC and a persistent workload.

The two consumers share capacity. Pool quotas are 6 GiB native and 10 GiB k3s; they are not extra physical space. Ceph storage HA is not application HA: each client is still a single VM, and all VMs run on one host computer.

## Files and records

See the [documentation guide](../README.md) for how docs and findings are kept. Experiment records live in [findings/](findings/2026-11-08-vagrant-vms.md), including what the VMs reported when checked. Facts live in the shared [knowledge bundle](../../knowledge/okf/index.md). See the [decisions](adr/README.md).

| File | Purpose |
| --- | --- |
| [vm/vagrant/podman/Vagrantfile](../../vm/vagrant/podman/Vagrantfile) | VMs, disks and addresses |
| [configs/podman/versions.yml](../../configs/podman/versions.yml) | Pinned images and version record |
| [configs/podman/cluster-spec.example.yaml](../../configs/podman/cluster-spec.example.yaml) | Hosts, labels, MON, MGR and OSD layout |
| [configs/podman/monitoring-spec.yaml](../../configs/podman/monitoring-spec.yaml) | Monitoring stack and HTTPS gateway |
| [configs/podman/clients/](../../configs/podman/clients) | Files for the optional client VMs |
| [ansible/vars/podman.yml](../../ansible/vars/podman.yml) | Ansible settings for this method |

## Current state

On 2026-11-09 the lab was rebuilt twice on new VMs: first by hand from the guides, then with Ansible. The running cluster is the Ansible-built one (fsid `73110d94-dbfb-11f1-9010-08002708f50c`), on the box kernel 7.0.0-34 without an OS upgrade. It is `HEALTH_OK`: three MONs, three MGRs and three OSDs on the `/dev/sdb` data disks, 60 GiB raw. The reserve disks are untouched. The dashboard, monitoring and an HTTPS gateway on each storage VM run, and the UI opens at `https://192.168.56.11/`, `.12/` or `.13/` from the host computer. Prometheus, Alertmanager and Grafana are under `/prometheus/`, `/alertmanager/` and `/grafana/` on each address; the first two ask for the login from `ceph orch prometheus get-credentials`. Dashboard POST requests through the gateway can fail with `503 Keep on looking`; see [ADR 0011](adr/0011-gateway-template-override.md). See the [end-to-end finding](findings/2026-11-09-end-to-end.md) and the [first finding](findings/2026-11-08-vagrant-vms.md). The first dashboard login has not been done on this cluster. The target configuration follows production practice where a laptop lab allows, but it is **not** production ready; [ADR 0010](adr/0010-production-aligned-baseline.md) lists the gaps.

## Index

- [VM learning path](diataxis/tutorials/vm-learning-path.md)
- [Create the VMs with Vagrant](diataxis/how-to/create-vagrant-vms.md)
- [Preflight checks](diataxis/how-to/vm-preflight.md)
- [Deploy Ceph by hand](diataxis/how-to/deploy-vm-cluster.md)
- [Enable and access the dashboard](diataxis/how-to/dashboard.md)
- [Deploy with Ansible](diataxis/how-to/deploy-with-ansible.md)
- [Fill the cluster and add the reserve disks](diataxis/how-to/add-reserve-disks.md)
- [Native VM RBD client](diataxis/how-to/native-rbd-client.md)
- [k3s RBD client](diataxis/how-to/k3s-rbd-client.md)
- [VM lab reference](diataxis/reference/vm-lab.md)
- [Validation matrix](diataxis/reference/vm-validation.md)
- [Design explanation](diataxis/explanation/vm-design.md)
- [Ceph HA and usable capacity](diataxis/explanation/ha-and-capacity.md)
- [Finding 2026-11-08: Vagrant VMs created and checked](findings/2026-11-08-vagrant-vms.md)
- [Finding 2026-11-09: End-to-end test, manual install then Ansible](findings/2026-11-09-end-to-end.md)
- [Decisions](adr/README.md)

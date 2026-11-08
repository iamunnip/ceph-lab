# VM learning path

Status: VMs created and checked on 2026-11-08. The Ceph lessons are planned.

1. [Create the five VMs with Vagrant](../how-to/create-vagrant-vms.md) and explain which disks Ceph may use.
2. [Check the VMs](../how-to/vm-preflight.md) before installing anything.
3. [Deploy three-host Ceph by hand with cephadm](../how-to/deploy-vm-cluster.md). Inspect MON quorum, MGR roles, OSD placement, pools and CRUSH.
4. [Enable the Ceph dashboard](../how-to/dashboard.md) and open it from your computer; inspect hosts, OSDs and health.
5. [Fill the cluster until it warns about low space, then add the reserve disks](../how-to/add-reserve-disks.md) and watch backfill.
6. Use an RBD image from a [normal Linux VM](../how-to/native-rbd-client.md) on node-04. Write a known dataset and prove persistence across remount and reboot.
7. Connect [single-node k3s](../how-to/k3s-rbd-client.md) on node-05 to Ceph using Ceph-CSI. Prove persistence across pod recreation.
8. Automate steps 3 to 5 with Ansible, once the manual steps are verified.
9. Add CephFS and compare shared filesystem access with RBD block storage.
10. Observe a controlled failure, for example `vagrant halt node-03`, explain the health output, recover and verify the original data.

Each lesson must include prerequisites, pinned artifacts, commands, expected output, success criteria, recovery and linked evidence. Complete the validation matrix before calling the VM track tested.

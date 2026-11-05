# VM learning path

Status: curriculum plan; executable walkthroughs will be added with implementation.

1. Inventory the three-VM lab on your chosen platform and explain which disks can be used safely.
2. Deploy three-host Ceph with cephadm and inspect MON quorum, MGR roles, OSD placement, pools and CRUSH.
3. Enable the Ceph dashboard and open its HTTPS URL; inspect hosts, OSDs and health.
4. As an optional later extension, use an RBD image from a native Linux service. Write a known dataset and prove persistence across remount and reboot.
5. Connect single-node k3s to external Ceph using Ceph-CSI. Provision a PVC with an explicit Ceph StorageClass and prove persistence across pod recreation.
6. Add CephFS and compare shared filesystem access with RBD block storage.
7. Observe a controlled failure, explain the health output, recover the failed component and verify the original data.
8. Capture diagnostics and rehearse a documented cleanup in a disposable lab.

Each implemented lesson must include prerequisites, pinned artifacts, commands, expected output, success criteria, recovery and linked experiment evidence. Complete the validation matrix before calling the VM track tested.

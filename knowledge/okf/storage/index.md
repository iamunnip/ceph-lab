# Disk

* [OS disk](os-disk.md) - Disk 1 in every VM. The box's 10 GB disk grown to 20 GB. Ubuntu boots from it; Ceph never uses it.
* [Data disk](data-disk.md) - Disk 2 in every storage VM. 20 GB, `/dev/sdb`, becomes the first OSD.
* [Reserve disk](reserve-disk.md) - Disk 3 in every storage VM. 20 GB, `/dev/sdc`, kept empty until the low-space exercise.

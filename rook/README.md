# Rook track

Status: deferred until the VM baseline is reproducible.

This track will explore Rook-managed Ceph. Its plan must explicitly choose between an operator-managed in-cluster Ceph deployment and Rook external-cluster integration, define device ownership and resource requirements, and compare lifecycle/recovery behavior with the cephadm baseline. Do not hand shared OSD devices to multiple cluster managers.

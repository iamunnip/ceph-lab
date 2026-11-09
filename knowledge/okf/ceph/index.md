# Ceph

* [Cluster](cluster.md) - Ceph 20.2.5 on node-01 to node-03, podman method. Deployed by hand on 2026-11-08, `HEALTH_OK`.
* [Daemons](daemons.md) - 3 MONs, 3 MGRs and 3 OSDs, placed by host label.
* [Pools](pools.md) - Replicated size 3, min_size 2. `.mgr`, `lab-smoke`, and the planned client pools.
* [Dashboard](dashboard.md) - The management UI in the active MGR, certificate from the cephadm CA.
* [Gateway](gateway.md) - The mgmt-gateway on every storage VM, HTTPS on port 443.
* [Monitoring](monitoring.md) - Prometheus, Alertmanager, Grafana and the exporters, all pinned by digest.

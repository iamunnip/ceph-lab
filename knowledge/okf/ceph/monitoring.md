---
type: Ceph Service
title: Monitoring
description: Prometheus, Alertmanager and Grafana on node-02, node-exporter and ceph-exporter on every storage VM. All images pinned by digest.
tags: [ceph, monitoring, prometheus, grafana]
resource: ../../../configs/podman/monitoring-spec.yaml
status: stable
sources:
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Service         | Version | Placement                 |
|-----------------|---------|---------------------------|
| Prometheus      | 3.6.0   | 1, label `monitoring` (node-02) |
| Alertmanager    | 0.28.1  | 1, label `monitoring`   |
| Grafana         | 12.3.1  | 1, label `monitoring`   |
| node-exporter   | 1.9.1   | every host                |
| ceph-exporter   | Ceph's  | every host                |

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Image pins         | `mgr/cephadm/container_image_<name>`, values in `configs/podman/versions.yml` |
| Retention          | 15 days and 2 GB                                           |
| Login              | Prometheus and Alertmanager answer `401` and ask for the login from `ceph orch prometheus get-credentials`. Grafana needs none |
| Direct ports       | 9095 and 9093 accept only TLS clients with a cephadm CA certificate |
| Receivers          | None. Alerts show only in the dashboard                    |
| Data on 2026-11-08 | 6.4M in Prometheus after about an hour                     |

# Rules

- `ceph orch stop prometheus` and `start prometheus` pause it without losing data. While stopped, alert rules and graphs stop; `ceph health` still works.
- `CephNodeDiskspaceWarning` fired in the first hour from the image-download trend, a false alarm[^finding-2026-11-08].

[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

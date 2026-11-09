---
type: Ceph Service
title: Gateway
description: The cephadm mgmt-gateway, nginx on HTTPS port 443 of every storage VM. Sends requests to the active MGR and serves the monitoring tools.
tags: [ceph, gateway, nginx, tls, dashboard]
resource: ../../../configs/podman/monitoring-spec.yaml
status: stable
sources:
  - id: adr-0010
    resource: ../../../docs/podman/adr/0010-production-aligned-baseline.md
    title: "ADR 0010: Align the VM deployment with production practice"
  - id: adr-0011
    resource: ../../../docs/podman/adr/0011-gateway-template-override.md
    title: "ADR 0011: mgmt-gateway template override for dashboard POST requests (tried and reverted)"
  - id: finding-2026-11-08
    resource: ../../../docs/podman/findings/2026-11-08-vagrant-vms.md
    title: "Finding 2026-11-08: Vagrant VMs created and checked"
---

# Facts

| Field              | Value                                                      |
|--------------------|------------------------------------------------------------|
| Service            | `mgmt-gateway`, placement `label: gateway`, on node-01 to node-03 |
| Image              | `quay.io/ceph/nginx` 1.26.1, pinned by digest             |
| Port               | 443, TLSv1.2 and TLSv1.3. Internal port 29443 for Prometheus and Alertmanager |
| URLs               | `https://192.168.56.11/`, `.12/`, `.13/`; `/grafana/`, `/prometheus/`, `/alertmanager/` |
| Certificate        | One self-signed `CN=ceph-lab-gateway` for `DNS:node-01..03` and `IP:192.168.56.11..13`, 825 days, stored as `mgmt_gw_cert` and `mgmt_gw_key` |
| Virtual IP         | None. Switching address after a VM failure is manual[^adr-0010] |

# Rules

- Needs the [host name lines](../networks/host-names.md): upstreams are `node-0N:8443`.
- The Ansible check needs, on every gateway: `CN=ceph-lab-gateway`, at least 30 days left (`openssl x509 -checkend 2592000`), all six SAN entries, and the same SHA-256 fingerprint. A failure makes a new certificate.
- After `ceph orch reconfig mgmt-gateway`, `ceph orch restart mgmt-gateway` is needed before nginx serves a new certificate.

# Known issues

- Dashboard POST requests can fail with `503 Keep on looking`: nginx does not retry a POST on the next MGR. A `non_idempotent` override fixed it in a test but was reverted[^adr-0011]. Workaround: open the active MGR from `ceph mgr services`.
- Right after a MGR failover the gateway answers `502` for about 20 seconds.

[^adr-0010]: ADR 0010: Align the VM deployment with production practice
[^adr-0011]: ADR 0011: mgmt-gateway template override for dashboard POST requests (tried and reverted)
[^finding-2026-11-08]: Finding 2026-11-08: Vagrant VMs created and checked

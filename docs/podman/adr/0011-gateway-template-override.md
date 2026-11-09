# 0011: mgmt-gateway template override for dashboard POST requests

Status: tried and reverted at the user's request. Not in effect. Date: 2026-11-08.

## Context

[ADR 0010](0010-production-aligned-baseline.md) puts the dashboard behind cephadm's `mgmt-gateway`. cephadm lists every manager as a dashboard backend. Standby managers answer `503 Keep on looking`, and the gateway's `proxy_next_upstream ... http_503` rule is meant to skip them.

nginx never passes a non-idempotent request, such as POST, to the next backend once it has been sent, unless `non_idempotent` is set. The dashboard uses POST for its login check and for actions. After a manager failover on 2026-11-08, six test POSTs through the gateway returned `415 503 503 415 415 503`: two in three reached a standby. GETs were fine. Before the dashboard had a certificate, the standbys did not listen at all; a refused connection is retried even for POST, which hid the problem at first.

The template is the same in Ceph v20.2.5 and in `main` on 2026-11-08. cephadm uses a template from its config store, `mgr/cephadm/<template path without .j2>`, instead of the built-in one when it exists.

## What was tried

A copy of the v20.2.5 `services/mgmt-gateway/external_server.conf.j2` (sha256 `fec000edde5fba3db28f953601188f87c6a4a3ae2a4e237ceacbb59807cba444`, equal to upstream) with `non_idempotent` added to `proxy_next_upstream` on line 80, stored as `mgr/cephadm/services/mgmt-gateway/external_server.conf`, then `ceph orch reconfig mgmt-gateway`.

Result: six POSTs returned `415` six times (the active manager rejecting the empty test body) and GETs `200`. After a further manager failover to node-03, every request returned `502` at first and worked again about 20 seconds later, with all POSTs reaching the active manager.

## Decision

Reverted on 2026-11-08 at the user's request, with `ceph config-key rm mgr/cephadm/services/mgmt-gateway/external_server.conf` and `ceph orch reconfig mgmt-gateway`. The gateway uses cephadm's built-in template again. The lab copy of the template was removed from the repository.

## Consequences

- Dashboard POST requests through `https://192.168.56.12/` can fail with `503 Keep on looking` again, depending on which manager nginx picks. The workaround is the active manager's own address from `ceph mgr services`, which changes on failover.
- No forked upstream template to maintain across Ceph upgrades.
- Open: choose a lasting fix, for example reapplying this override, a different cephadm setting, or an upstream report.

## References

- [Known issue in the deployment guide](../diataxis/how-to/deploy-vm-cluster.md#known-issue-dashboard-post-requests-can-fail-with-503)
- [VM finding](../findings/2026-11-08-vagrant-vms.md)
- [nginx proxy_next_upstream](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)

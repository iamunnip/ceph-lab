# 0002: VM cluster topology and cephadm

Status: accepted as initial lab design; resource fit remains a preflight gate. Date: 2026-11-05.

## Context

Five iximiuz Labs VMs already communicate through WireGuard hub-and-spoke networking. Three have an additional 80 GB disk each; two are clients.

## Decision

Use cephadm on three storage VMs, one explicitly selected OSD device and one MON per storage host, plus active/standby MGR on separate hosts. Start with replicated pools using size 3, min_size 2 and host failure domains. Reuse existing WireGuard networking without provisioning or changing it.

## Consequences

The lab can demonstrate one-host failure behavior, but cannot restore three replicas while one host is absent. The existing hub is a shared network dependency. Memory, platform limits, eligible disks and exact supported versions must be established before deployment. Device discovery never authorizes wiping devices.

## Implementation follow-up

The initial topology was deployed and validated on 2026-11-05. [ADR 0004](0004-vm-deployment-baseline.md) records concrete hosts, artifacts and operational choices; the [deployment finding](../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) records measured results and remaining validation.

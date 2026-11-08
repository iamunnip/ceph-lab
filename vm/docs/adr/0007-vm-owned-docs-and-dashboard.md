# 0007: Scope VM documentation to three storage hosts and the dashboard

Status: accepted. Date: 2026-11-05, revised 2026-11-08.

## Context

Each track keeps its own material. The core VM walkthrough should need only the three storage VMs, each using its data disk, with the Ceph UI enabled.

## Decision

Keep VM documentation in `vm/docs/` and knowledge in `vm/knowledge/`, using Diataxis, ADRs, arc42 and the knowledge register inside the track.

The core setup requires exactly three storage hosts with a separate OS and data disk, and the dashboard. Client nodes 04 and 05 are optional extensions. Enable the dashboard with HTTPS, a self-signed lab certificate, and an administrator whose initial password must be changed. Open it from the host computer over the host-only network; see [ADR 0010](0010-production-aligned-baseline.md) for the gateway. Never expose MON or OSD ports to reach the dashboard.

## Consequences

Readers can complete the core setup with three VMs. Passwords and TLS private keys stay outside the repository.

This changes the documentation placement in ADR 0001 and makes the client stages in ADR 0003 extensions beyond the core setup.

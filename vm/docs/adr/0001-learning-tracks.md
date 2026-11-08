# 0001: Separate learning tracks and maintain documentation with changes

Status: accepted. Date: 2026-11-05.

## Context

The repository explores Ceph on VMs, Kubernetes and Rook. Implementation starts with the VM track, and documentation and knowledge must stay current.

## Decision

Keep `vm/`, `k8s/` and `rook/` as separate track roots. Organize task documentation with Diataxis, decisions with ADRs, and the architecture overview with arc42 and C4-style diagrams. Maintain open questions, known facts and experiment findings under `knowledge/okf/`.

## Consequences

Every meaningful implementation/experiment change includes corresponding documentation and knowledge updates. Future tracks must define whether they consume an external cluster or deploy their own; directory names alone do not establish that boundary.

## Scope follow-up

[ADR 0007](0007-vm-owned-docs-and-dashboard.md) moves VM docs/knowledge under the VM track, makes client stages optional extensions, and enables the dashboard in the three-host core setup.

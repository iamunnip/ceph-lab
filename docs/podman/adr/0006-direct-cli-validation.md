# 0006: Use direct Ceph commands for lab verification

Status: accepted. Date: 2026-11-05.

## Context

The initial baseline used a Python validator and a shell wrapper. The current learning stage can be checked using standard Ceph commands, and the user requested removal of both helpers.

## Decision

Remove both custom scripts. Keep direct Ceph CLI commands, explanations and expected results in the [deployment guide](../diataxis/how-to/deploy-vm-cluster.md#verification-and-recovery). Inventory and pinned-version JSON files remain reference records and are not prerequisites for the manual walkthrough.

## Consequences

Readers inspect health, daemon placement and pool policy directly. Automated validation can be reconsidered if repeatable deployments or CI establish a concrete need. This supersedes only the custom-validator and transport decisions in [ADR 0004](0004-vm-deployment-baseline.md) and [ADR 0005](0005-provider-independent-guides.md); their other decisions remain in effect.

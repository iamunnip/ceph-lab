# 0005: Make VM guides independent of the provider

Status: accepted. Date: 2026-11-05.

## Context

The first deployment used iximiuz Labs and labctl. The project is intended for everyone with suitable Linux VMs, including readers using other platforms. Main guides previously embedded our playground ID, SSH account and mesh addresses and assumed that the test pool already existed.

## Decision

Use standard SSH and clearly labeled example hostnames, addresses, usernames and disk paths in the main VM guide. Explain substitutions and prerequisites. Keep provider provisioning and network setup outside the deployment steps; require existing routed connectivity without requiring WireGuard.

Keep iximiuz access details in an optional provider guide and preserve observed deployment inventory/findings as evidence. Provide a generic JSON inventory template. Require the validator's inventory argument and default its transport to SSH, with an explicit labctl option. Private local inventory is ignored by Git. Ubuntu package commands are labeled as distribution-specific rather than claimed to work on every Linux distribution.

Walkthroughs identify the execution host, explain commands and expected output, and include creation of resources needed for a fresh run. Python validators remain allowed.

## Consequences

Readers need neither an iximiuz account nor labctl to follow the main guide. They must supply VMs, suitable disposable disks, supported host software and working networking. Other providers have not been live-tested by this change; portability is supported by the documented contract and SSH transport, not a claim of validation on every platform.

This refines [ADR 0004](0004-vm-deployment-baseline.md): its concrete deployment choices remain historical evidence, while its provider-specific access is no longer the default reader path.

## References

- [Main guide](../diataxis/how-to/deploy-vm-cluster.md)
- [Optional iximiuz access](../diataxis/how-to/iximiuz-access.md)
- [Generic inventory](../../inventory/example.json)

## Follow-up

[ADR 0006](0006-direct-cli-validation.md) supersedes the custom-validator decisions above. The scripts have been removed; current walkthroughs use direct Ceph CLI commands.

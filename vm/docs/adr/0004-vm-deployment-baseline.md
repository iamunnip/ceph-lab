# 0004: Pin and operate the initial VM deployment

Status: accepted and implemented. Date: 2026-11-05.

## Context

[ADR 0002](0002-vm-topology.md) selected the three-host cephadm topology. Live inventory found five Ubuntu 26.04.1 VMs with 4 vCPUs, 10024 MiB RAM, kernel 6.1.167, 20 GiB root devices and approximately 80 GiB additional disks. All playgrounds use the same eth0 address, while existing mesh0 addresses are unique.

## Decision

- Map storage hosts to node-01/02/03 at 10.8.0.1/2/3. Reserve node-04 for native clients and node-05 for k3s. Use existing mesh addresses for Ceph administration and data traffic; leave WireGuard unchanged.
- Pin Ceph and standalone cephadm to 20.2.5 and pin the Ceph container by digest. Use Podman 5.7.0 from the host distribution. Record exact values in [versions.json](../../config/versions.json); this lab's successful execution does not imply upstream certification of every host component.
- Keep observed machine facts in [inventory/iximiuz.json](../../inventory/iximiuz.json). Keep selected artifact versions separate from inventory. JSON is directly consumable by the Python validator and requires no additional parsing dependency. Beginner walkthroughs use ordinary shell commands.
- After explicit user approval and empty-content/identity checks, repurpose only `/dev/vdb` on nodes 01–03 from `/data` ext4 filesystems into LVM-backed BlueStore OSDs. Preserve root and client disks.
- Disable OSD memory autotuning and set the per-OSD memory target to 4 GiB to leave room for co-located daemons. This is not a hard memory limit.
- Defer dashboard and monitoring deployment. Use Ceph CLI health, daemon logs and a Python baseline validator initially. Install logrotate and explicitly start Chrony/SSH where the base image suppresses package service startup.
- Retain a small `lab-smoke` RADOS pool/object for inspecting a write/read test. Use size 3/min_size 2 and host-level CRUSH placement, consistent with ADR 0002.

## Consequences

The deployed baseline is HEALTH_OK with three MONs, two MGRs and three OSDs. A write/read checksum test and seven baseline validator checks passed. No scoped application-client, Kubernetes, full-host failure or reboot persistence claim follows from those checks.

The cluster depends on existing mesh routing and playground lifetime. Recovery memory needs and future MDS placement must be measured before expanding scope. Client versions remain unset until their implementation stage. Credentials remain on VMs, outside the repository.

## Evidence and operations

See the [deployment finding](../../knowledge/okf/findings/2026-11-05-vm-bootstrap.md) for disk identities, SSH newline troubleshooting, transient convergence, manager promotion and validation. See the [beginner guide](../diataxis/how-to/deploy-vm-cluster.md) for commands and expected output.

This ADR refines ADR 0002 without superseding its topology decision.

## Follow-up

[ADR 0006](0006-direct-cli-validation.md) supersedes the custom-validator decisions above. The scripts have been removed; current walkthroughs use direct Ceph CLI commands.

## Scope follow-up

[ADR 0007](0007-vm-owned-docs-and-dashboard.md) moves VM docs/knowledge under the VM track, makes client stages optional extensions, and enables the dashboard in the three-host core setup. Earlier deployment observations remain historical evidence.

## Follow-up on 2026-11-05 redeployment

[ADR 0010](0010-production-aligned-baseline.md) deploys the monitoring stack, removes the retained `lab-smoke` pool and uses a declarative spec. The observations above describe the earlier playgrounds.

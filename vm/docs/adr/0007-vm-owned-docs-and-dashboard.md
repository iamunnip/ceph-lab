# 0007: Scope VM documentation to three storage hosts and the dashboard

Status: accepted and implemented. Date: 2026-11-05.

## Context

VM-specific docs and knowledge were stored at repository root. The user wants each setup's material in its own track and a core VM walkthrough requiring only three VMs, each using its second disk, with the Ceph UI enabled. The current platform is iximiuz Labs, while other readers may use VirtualBox or other providers.

## Decision

Move existing documentation to `vm/docs/` and knowledge to `vm/knowledge/`. Keep Diataxis, ADRs, arc42 and OKF inside that track. Preserve prior ADRs and findings as history and repair links after the move.

The core VM setup requires exactly three storage hosts with a separate OS and data disk. Existing client nodes 04–05 are optional later extensions. Enable the dashboard module with HTTPS, a self-signed lab certificate, and an administrator whose initial password must be changed. Monitoring charts remain deferred.

For VirtualBox and other reachable VM platforms, open the active manager's HTTPS address directly. For iximiuz, bind the dashboard on the VM interfaces and use `labctl expose port`. The initial HTTPS-backend exposure failed because the ingress handshake could not negotiate the TLS-1.3-only Ceph backend. The user explicitly approved a lab-only HTTP backend on 18080 exposed with `labctl expose port <active-manager-playground> 18080 --public`; the public URL remains HTTPS. Other providers retain direct dashboard HTTPS on 8443. Do not introduce port forwarding or an additional proxy service. Never expose MON/OSD ports as a way to reach the dashboard.

## Consequences

Readers can complete the core setup with three VMs. Provider-specific access remains in its own guide. Manager failover may require exposing the new active manager's playground. A publicly reachable UI still requires dashboard authentication. Passwords and TLS private keys stay outside the repository.

This changes the documentation placement in ADR 0001, defers the client stages in ADR 0003 beyond the core setup, and supersedes the dashboard deferral in ADR 0004. Their unrelated decisions remain in effect.

## Evidence

See the [dashboard finding](../../knowledge/okf/findings/2026-11-05-dashboard.md), [dashboard guide](../diataxis/how-to/dashboard.md), and [iximiuz access guide](../diataxis/how-to/iximiuz-access.md).

## Follow-up on 2026-11-05 redeployment

[ADR 0010](0010-production-aligned-baseline.md) replaces the HTTP backend on 18080, the no-proxy rule and `--public` exposure with the cephadm `mgmt-gateway` on HTTPS port 443, exposed privately.

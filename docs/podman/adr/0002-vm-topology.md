# 0002: VM cluster topology and cephadm

Status: accepted; deployed and tested end to end on 2026-11-09, see the [finding](../findings/2026-11-09-end-to-end.md). Date: 2026-11-05, revised 2026-11-08.

## Context

The lab has five VMs: three for Ceph storage and two clients. Each storage VM has a separate OS disk and a dedicated data disk. See [ADR 0009](0009-vagrant-virtualbox-lab.md) for the platform.

## Decision

Use cephadm on three storage VMs, one explicitly selected OSD device per host to start, one MON per storage host, and three MGRs (one active, two standby). Start with replicated pools using size 3, min_size 2 and host failure domains. Use one Ceph public network, the host-only network 192.168.56.0/24, and no separate cluster network.

## Alternatives considered on 2026-11-08

| Option | Why not |
| --- | --- |
| cephadm with Docker | Works, but adds a daemon under every Ceph container and a package outside the pinned set; upstream recommends Podman |
| Manual package install | Ubuntu 26.04 offers only Ceph 20.2.0, and upstream 20.2.5 packages exist only for Debian 12 and Ubuntu 22.04 and 24.04. No orchestrator, so no specs, mgmt-gateway or `ceph orch upgrade`. Kept as a possible extra exercise to learn the internals. |
| MicroCeph snap | Ships 20.2.1 and hides most deployment steps the lab means to teach |
| ceph-ansible, ceph-deploy | No longer maintained for current releases |

cephadm runs every daemon as a container from the digest-pinned 20.2.5 image under Podman 5.7.0 from Ubuntu.

## Consequences

The lab can demonstrate one-host failure behavior, but cannot restore three replicas while one host is absent. All VMs share one host computer, so host failure domains are logical, not physical. Device discovery never authorizes wiping devices.

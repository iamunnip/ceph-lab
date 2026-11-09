# 0003: External clients and storage progression

Status: accepted as initial lab design. Date: 2026-11-05.

## Context

Two client VMs should demonstrate ordinary services and Kubernetes workloads using Ceph storage.

## Decision

Dedicate one client to native Linux consumption and one to single-node k3s with Ceph-CSI. Begin with RBD, add CephFS with active/standby MDS, and defer RGW/S3 to an optional extension. Use scoped identities and explicit StorageClasses. The VM track's Ceph cluster remains managed by cephadm.

## Consequences

Kubernetes storage integration can be learned independently of Rook. Single-node k3s cannot demonstrate Kubernetes node failover or multi-node attachment behavior. RWX pods on that VM show filesystem semantics, not multi-node Kubernetes availability. CSI versions and privileged node-plugin requirements must be checked before deployment.

## Scope follow-up

[ADR 0007](0007-vm-owned-docs-and-dashboard.md) moves VM docs/knowledge under the VM track, makes client stages optional extensions, and enables the dashboard in the three-host core setup.

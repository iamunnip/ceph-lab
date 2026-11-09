# 0005: Use Vagrant lab values in the guides, with ordinary SSH between VMs

Status: accepted. Date: 2026-11-05, revised 2026-11-08.

## Context

Readers should be able to copy and paste each command. Guides full of placeholders are harder to follow, while every reader can create the same VMs from the Vagrantfile.

## Decision

Main guides use the Vagrant lab's real values: user `vagrant`, addresses 192.168.56.11 to 15, data disk `/dev/sdb`, reserve disk `/dev/sdc`. Readers connect with `vagrant ssh` and copy files with `vagrant upload`. Ceph itself uses ordinary SSH between the VMs, as cephadm requires. Each guide says once which values to change if the VMs come from somewhere else.

Walkthroughs identify the VM that runs each command, explain commands and expected output, and include creation of resources needed for a fresh run. Ubuntu package commands are labeled as distribution-specific.

## Consequences

Vagrant readers can follow the guides without edits. Readers on other platforms must supply VMs with equivalent disks and networking and substitute their values. Other platforms are not tested.

See [ADR 0006](0006-direct-cli-validation.md) for verification without custom scripts.

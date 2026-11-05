# Open questions, known facts and findings

Updated: 2026-11-05. This register separates user-supplied facts, planned decisions, and observed evidence. Live inventory and initial deployment evidence are linked below.

## Known facts

| ID | Fact | Source |
| --- | --- | --- |
| K001 | Five iximiuz Labs VMs are available | User |
| K002 | WireGuard hub-and-spoke setup is already complete; setup is out of scope | User |
| K003 | Three storage VMs each have an additional 80 GB disk | User |
| K004 | Two VMs are clients, including a service consumer and a k3s storage consumer | User |
| K005 | VM work comes first; docs and knowledge must remain updated | User |
| K006 | Original repository code, configurations and documentation use the MIT License; third-party material retains its original terms | User-approved choice; [LICENSE](../../../LICENSE) |
| K007 | Three-node Ceph baseline is HEALTH_OK; write/read and seven validator checks passed | [Deployment finding](findings/2026-11-05-vm-bootstrap.md) |
| K008 | Use beginner-friendly documentation and direct Ceph commands; custom validator and shell wrapper removed at user request | User; [ADR 0006](../../docs/adr/0006-direct-cli-validation.md) |
| K009 | Guides must work for readers using other VM platforms; standard SSH is the default and iximiuz access is optional | User; [ADR 0005](../../docs/adr/0005-provider-independent-guides.md) |
| K010 | VM docs and knowledge belong under vm/; core setup needs three VMs, second disks and the dashboard; iximiuz UI access uses labctl expose | User; [ADR 0007](../../docs/adr/0007-vm-owned-docs-and-dashboard.md) |
| K011 | Native node-04 and k3s node-05 consume separate RBD images from the same replicated capacity; combined logical pool budget is 60 GiB | [ADR 0008](../../docs/adr/0008-rbd-clients-and-capacity-budget.md) |

## Open questions

| ID | Question / evidence needed | Resolution stage |
| --- | --- | --- |
| Q001 | Resolved: observed inventory and synchronized storage-host clocks recorded | [Finding](findings/2026-11-05-vm-bootstrap.md) |
| Q002 | Partial: healthy idle baseline on 4 vCPU/10024 MiB hosts, about 16 GiB root free; recovery and MDS resource budgets remain untested | Recovery experiments |
| Q003 | Resolved for nodes 01–03: serials inspected, empty disks approved and provisioned as OSDs | [Finding](findings/2026-11-05-vm-bootstrap.md) |
| Q004 | Partial: both clients reach all MON/OSD endpoints; MTU under load and full firewall analysis remain open | Network/load experiments |
| Q005 | Partial: Ceph/cephadm 20.2.5 and Podman 5.7.0 validated at baseline; k3s v1.36.6+k3s1 and Ceph-CSI 3.18.2 deployed | Client implementation |
| Q006 | Which playground operations preserve or destroy disks, identities and IPs? | Preflight |
| Q007 | Resolved for NBD lab mapping: both clients load NBD; CSI privileged pods run with /var/lib/kubelet; kernel Ceph key compatibility remains a limitation. | Client preflight |
| Q008 | What is the hub's location and resulting failure blast radius? | Inventory; no WireGuard changes |
| Q009 | Resolved: native nginx serves files from a 10 GiB Ceph image; remount persistence tested. | Native-client implementation |

## Decisions and implementation status

Deployed: cephadm 20.2.5, three MONs and OSDs, active/standby MGR, size 3/min_size 2 host-replicated pools. [ADR 0004](../../docs/adr/0004-vm-deployment-baseline.md) records pinned artifacts, inventory/config formats, disk selection, memory target and deferred monitoring. Measured results are in the findings below.

Dashboard is enabled with an administrator; direct VM guidance retains HTTPS, while the approved iximiuz backend is HTTP on 18080 behind public HTTPS; access evidence is recorded in the dashboard finding. Native RBD on node-04 and k3s RBD on node-05 are now implemented extensions. CephFS remains deferred. These clients are not additional OSD hosts.

## Documentation portability

The main VM guide now uses standard SSH, replaceable example values and fresh test-pool creation steps. A generic inventory template is available as a reference; manual commands do not require JSON inventory. iximiuz details remain in a provider guide and observed finding. See [ADR 0005](../../docs/adr/0005-provider-independent-guides.md). Validation: all seven checks passed against the existing cluster using explicit labctl transport. A mocked run verified ordinary SSH command construction with the generic inventory and all checks passed. Markdown links and shell syntax checks passed. Other providers are not live-tested yet. The transport tests above are historical: the custom validator and shell wrapper have since been removed. Current checks use direct Ceph CLI commands, as recorded in [ADR 0006](../../docs/adr/0006-direct-cli-validation.md).

## Latest HA result

With node-03 MON/OSD stopped, both native and k3s clients wrote and read data with two-replica availability. After restoration, health returned to HEALTH_OK with 97 active+clean PGs. This is a controlled Ceph-service failure test, not a full VM power-loss or application-failover claim. See the client finding.

## Findings

- [2026-11-05: HA, capacity and native/k3s clients](findings/2026-11-05-client-storage.md).

- [2026-11-05: Dashboard enablement and exposure](findings/2026-11-05-dashboard.md).

- [2026-11-05: VM inventory and cluster deployment](findings/2026-11-05-vm-bootstrap.md) (healthy baseline and write/read test passed).

Add a finding per experiment with: ID/date; linked question or hypothesis; exact versions/topology; procedure; expected and actual result; sanitized evidence; limitations; recovery result; and documentation/decision changes. Mark failed and inconclusive experiments honestly. Never store credentials in evidence.

## Local credential files

The repository ignores local inventories, environment files, cloud/Kubernetes credentials, Ceph keyrings, private-key/PEM files and runtime logs. Example environment files remain trackable. These rules reduce accidental additions; review staged files for secrets before committing. Existing tracked files are not protected by ignore patterns.

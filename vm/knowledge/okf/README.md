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
| K012 | Standing approval to wipe `/dev/vdb` on iximiuz node-01 to node-03 for new deployments when all checks pass; no approval for any other disk, host or provider | User, 2026-11-05; [ADR 0009](../../docs/adr/0009-standing-disk-wipe-approval.md) |
| K013 | Kernel RBD fails on kernel 6.1.167 with current Ceph keys (`libceph: secret too big 32`); RBD-NBD stays until the kernel changes | [Baseline finding](findings/2026-11-05-production-baseline.md) |
| K014 | node-01 is the WireGuard hub; rebooting it also separates the other nodes from each other | [Baseline finding](findings/2026-11-05-production-baseline.md) |
| K015 | Current deployment is production-aligned configuration on non-production infrastructure; gaps are listed in ADR 0010 | [ADR 0010](../../docs/adr/0010-production-aligned-baseline.md) |

## Open questions

| ID | Question / evidence needed | Resolution stage |
| --- | --- | --- |
| Q001 | Resolved: observed inventory and synchronized storage-host clocks recorded | [Finding](findings/2026-11-05-vm-bootstrap.md) |
| Q002 | Partial: healthy idle baseline on 4 vCPU/10024 MiB hosts, about 16 GiB root free; recovery and MDS resource budgets remain untested | Recovery experiments |
| Q003 | Resolved for nodes 01–03: serials inspected, empty disks approved and provisioned as OSDs | [Finding](findings/2026-11-05-vm-bootstrap.md) |
| Q004 | Partial: both clients reach all MON/OSD endpoints; mesh MTU measured at 1280; behavior under load and full firewall analysis remain open | Network/load experiments |
| Q005 | Partial: Ceph/cephadm 20.2.5 and Podman 5.7.0 validated at baseline; k3s v1.36.6+k3s1 and Ceph-CSI 3.18.2 deployed | Client implementation |
| Q006 | Partial: guest reboots preserve disks, identities and mesh addresses on all tested hosts; playground stop, start and expiry remain untested | Preflight |
| Q007 | Resolved: kernel RBD tested and blocked by key size on 6.1.167; both clients load NBD; CSI privileged pods run with /var/lib/kubelet. Which kernel first accepts these keys remains open. | Client preflight |
| Q008 | Partial: node-01 is the hub and a short reboot interrupts spoke-to-spoke traffic; effects of a long hub outage remain untested | Inventory; no WireGuard changes |
| Q009 | Resolved: native nginx serves files from a 10 GiB Ceph image; remount persistence tested. | Native-client implementation |

## Decisions and implementation status

Deployed: cephadm 20.2.5, three MONs, three MGRs, three OSDs, monitoring stack, HTTPS gateway, and size 3/min_size 2 host-replicated pools. Native RBD on node-04 and k3s RBD on node-05 are implemented extensions, not additional OSD hosts. CephFS remains deferred. [ADR 0010](../../docs/adr/0010-production-aligned-baseline.md) records the choices and gaps. Measured results are in the findings below.

## Documentation portability

The main VM guide now uses standard SSH, replaceable example values and fresh test-pool creation steps. A generic inventory template is available as a reference; manual commands do not require JSON inventory. iximiuz details remain in a provider guide and observed finding. See [ADR 0005](../../docs/adr/0005-provider-independent-guides.md). Validation: all seven checks passed against the existing cluster using explicit labctl transport. A mocked run verified ordinary SSH command construction with the generic inventory and all checks passed. Markdown links and shell syntax checks passed. Other providers are not live-tested yet. The transport tests above are historical: the custom validator and shell wrapper have since been removed. Current checks use direct Ceph CLI commands, as recorded in [ADR 0006](../../docs/adr/0006-direct-cli-validation.md).

## Latest HA result

With all node-03 Ceph services stopped, both clients wrote in under 0.25 s while all PGs were undersized, and HEALTH_OK returned 13 s after restoring services. A node-01 hub reboot caused one 27 s client write stall and no errors. This is not a power-loss or application-failover claim. See the [baseline finding](findings/2026-11-05-production-baseline.md).

## Findings

- [2026-11-05: Production-aligned redeployment and failure tests](findings/2026-11-05-production-baseline.md). Current deployment; the findings below describe the earlier playgrounds.

- [2026-11-05: HA, capacity and native/k3s clients](findings/2026-11-05-client-storage.md).

- [2026-11-05: Dashboard enablement and exposure](findings/2026-11-05-dashboard.md).

- [2026-11-05: VM inventory and cluster deployment](findings/2026-11-05-vm-bootstrap.md) (healthy baseline and write/read test passed).

Add a finding per experiment with: ID/date; linked question or hypothesis; exact versions/topology; procedure; expected and actual result; sanitized evidence; limitations; recovery result; and documentation/decision changes. Mark failed and inconclusive experiments honestly. Never store credentials in evidence.

## Local credential files

The repository ignores local inventories, environment files, cloud/Kubernetes credentials, Ceph keyrings, private-key/PEM files and runtime logs. Example environment files remain trackable. These rules reduce accidental additions; review staged files for secrets before committing. Existing tracked files are not protected by ignore patterns.

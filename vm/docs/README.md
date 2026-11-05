# VM documentation guide

Documentation and knowledge updates are part of every implementation or experiment, in the same change as the work.

- Tutorials explain a reproducible learning path and its expected outcomes.
- How-to guides explain focused tasks, prerequisites, verification, and recovery.
- Reference records exact configuration, versions, inventory, and acceptance criteria.
- Explanation describes concepts, constraints, and tradeoffs.
- ADRs record decisions and consequences; supersede accepted decisions rather than silently rewriting their history.
- arc42 records the system boundaries, components, runtime behavior, and risks.
- `vm/knowledge/okf/` records open questions, known facts, and measured findings. The directory name is retained as requested; no external OKF methodology is assumed.

For every meaningful change, update the affected guides/reference and the knowledge register. Record a new ADR when a consequential design choice changes. Link evidence to its experiment, date, environment, and exact versions. Distinguish planned behavior from verified behavior. Never commit credentials, private keys, tokens, admin keyrings, or unredacted kubeconfigs.

## Index

- [Production-aligned baseline decision](adr/0010-production-aligned-baseline.md)
- [Ceph HA and usable capacity](diataxis/explanation/ha-and-capacity.md)
- [Native VM RBD client](diataxis/how-to/native-rbd-client.md)
- [k3s RBD client](diataxis/how-to/k3s-rbd-client.md)

- [Enable and access the dashboard](diataxis/how-to/dashboard.md)

- [VM learning path](diataxis/tutorials/vm-learning-path.md)
- [Preflight procedure](diataxis/how-to/vm-preflight.md)
- [Optional iximiuz access](diataxis/how-to/iximiuz-access.md)
- [VM cluster deployment](diataxis/how-to/deploy-vm-cluster.md)
- [VM lab reference](diataxis/reference/vm-lab.md)
- [Validation matrix](diataxis/reference/vm-validation.md)
- [Design explanation](diataxis/explanation/vm-design.md)
- [Decisions](adr/README.md)
- [Architecture](arc42/README.md)

# Documentation guide

Documentation and knowledge updates are part of every implementation or experiment, in the same change as the work.

- Tutorials explain a reproducible learning path and its expected outcomes.
- How-to guides explain focused tasks, prerequisites, verification, and recovery.
- Reference records exact configuration, versions, inventory, and acceptance criteria.
- Explanation describes concepts, constraints, and tradeoffs.
- ADRs record decisions and consequences; supersede accepted decisions rather than silently rewriting their history.
- Findings (`docs/<method>/findings/`) are the full record of each experiment: date, versions, commands, expected and observed results, incidents and limitations.
- [knowledge/okf/](../knowledge/okf/index.md) is a facts-only OKF bundle in the same layout as kube-lab: an `index.md`, a `log.md` and one folder per topic, with one page per thing. Each page has frontmatter (`type`, `title`, `description`, `tags`, `resource`, `status`, `sources`), a Facts table and, where needed, Rules and Open questions. Pages cite ADRs and findings as sources.

For every meaningful change, update the affected guides/reference, the affected knowledge pages and `knowledge/okf/log.md`. Record a new ADR when a consequential design choice changes. Link evidence to its experiment, date, environment, and exact versions. Distinguish planned behavior from verified behavior. Never commit credentials, private keys, tokens, admin keyrings, or unredacted kubeconfigs.

## Tracks

Each install method has its own folder here. Its other files:

| Track | Docs | Files |
| --- | --- | --- |
| podman (cephadm with Podman) | [docs/podman/](podman/README.md) | [vm/vagrant/podman/](../vm/vagrant/podman/Vagrantfile), [configs/podman/](../configs/podman/versions.yml), [ansible/vars/podman.yml](../ansible/vars/podman.yml) |
| snap (Ceph from a snap, planned) | [docs/snap/](snap/README.md) | [vm/vagrant/snap/](../vm/vagrant/snap/Vagrantfile), [ansible/vars/snap.yml](../ansible/vars/snap.yml) |

Facts from every track go into the shared [knowledge bundle](../knowledge/okf/index.md); each track keeps its own findings.

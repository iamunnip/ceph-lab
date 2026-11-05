# ceph-lab

A hands-on lab for learning Ceph on virtual machines and Kubernetes, including Rook.

The VM track contains the deployment plan, observed inventory, pinned Ceph versions, and [live deployment evidence](vm/knowledge/okf/findings/2026-11-05-vm-bootstrap.md). Kubernetes and Rook deployment tracks are planned for later.

## Repository structure

```text
vm/                  # Three-VM Ceph setup and dashboard
  docs/
    diataxis/        # Tutorials, how-to, reference and explanation
    adr/             # VM track decisions
    arc42/           # VM architecture
  knowledge/okf/     # Questions, facts and experiment findings
  inventory/         # Example and observed host records
  config/            # Pinned versions
k8s/                 # Future Kubernetes track
rook/                # Future Rook track
```

The main walkthrough uses ordinary SSH and works with VMs from any provider; iximiuz Labs is an optional tested example.

Start with the [VM plan](vm/README.md), [architecture](vm/docs/arc42/README.md), and [knowledge register](vm/knowledge/okf/README.md).
See the [documentation guide](vm/docs/README.md) for how changes and experiment findings are recorded.

## License

Original code, configurations, and documentation are licensed under the [MIT License](LICENSE).
Third-party material retains its original license and notices.

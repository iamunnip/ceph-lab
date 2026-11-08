# ceph-lab

A hands-on lab for learning Ceph on virtual machines and Kubernetes, including Rook.

The VM track runs on Vagrant and VirtualBox. It contains the Vagrantfile, the manual deployment guide, observed inventory, pinned Ceph versions and [findings](vm/knowledge/okf/README.md). Ansible automation follows the manual install. Kubernetes and Rook deployment tracks are planned for later.

## Repository structure

```text
vm/                  # Three-VM Ceph setup and dashboard
  vagrant/           # Vagrantfile for the lab VMs
  docs/
    diataxis/        # Tutorials, how-to, reference and explanation
    adr/             # VM track decisions
    arc42/           # VM architecture
  knowledge/okf/     # Questions, facts and experiment findings
  inventory/         # Observed host records
  config/            # Pinned versions and cephadm specs
k8s/                 # Future Kubernetes track
rook/                # Future Rook track
```

Create the VMs with `vagrant up` from `vm/vagrant/`, then follow the guides. They use the lab's real addresses and disk paths, so commands can be copied as they are.

Start with the [VM plan](vm/README.md), [architecture](vm/docs/arc42/README.md), and [knowledge register](vm/knowledge/okf/README.md).
See the [documentation guide](vm/docs/README.md) for how changes and experiment findings are recorded.

## License

Original code, configurations, and documentation are licensed under the [MIT License](LICENSE).
Third-party material retains its original license and notices.

# Repository working rules

- Start with the VM track; Kubernetes and Rook deployment tracks remain deferred until explicitly planned.
- The VM lab runs on Vagrant and VirtualBox from `vm/vagrant/Vagrantfile`. Ceph uses only the host-only network 192.168.56.0/24 on `enp0s8`, never the NAT address 10.0.2.15.
- Install Ceph by hand first and verify each step. Use manual commands only: one plain command at a time, no scripts, loops, `sh -c` wrappers or helper files. Record every command in the docs with its purpose, the VM that runs it and its expected output. Ansible automation follows only after the manual steps are verified, and is built from those recorded commands.
- Never run or suggest npm, npx, pnpm, yarn, bun, node or other JavaScript tooling.
- Update affected `vm/docs/` and `vm/knowledge/okf/` alongside every meaningful implementation change or experiment. Separate planned behavior from observed evidence.
- Record consequential architecture changes in `vm/docs/adr/`; keep the architecture and track plans consistent.
- Never commit secrets or unredacted cluster credentials.
- Never infer permission to wipe a disk from its size or availability. Destructive operations need explicit targets, ownership checks and confirmation.
  New Vagrant disks are blank, so a deployment never needs a wipe; to start over, recreate the VMs. Never remove a disk from the Vagrantfile while its VM exists, since Vagrant then deletes it.
- Pin deployment artifacts and record compatibility evidence before installing components. Validate stage prerequisites and document recovery.

- Keep documentation beginner friendly: use separate, copy-and-paste shell commands, explain their purpose and expected output, and identify which VM runs them. Use standard Ceph CLI commands for current lab verification; do not reintroduce custom validators or command wrappers without a concrete automation need.

- Main guides use the Vagrant lab's real values (`vagrant` user, 192.168.56.11 to 15, data disk `/dev/sdb`, reserve disk `/dev/sdc`) and say once what to change for other platforms. Keep observed deployment details in inventories or findings.

- The core VM setup requires three storage VMs, their data disks and the Ceph dashboard, opened from the host at https://192.168.56.11/, .12/ or .13/ (a gateway runs on each storage VM). The reserve disk `/dev/sdc` joins Ceph only in the low-space exercise. Extra client VMs are optional extensions.

- Current client extensions consume RBD from node-04 (native service) and node-05 (single-node k3s/Ceph-CSI). Distinguish Ceph storage HA from client/application HA, and explain shared raw versus replicated usable space. Keep newer Ceph credentials; do not relax authentication to accommodate older clients.

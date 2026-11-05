# Repository working rules

- Start with the VM track; Kubernetes and Rook deployment tracks remain deferred until explicitly planned.
- WireGuard is already configured. Do not provision or change its topology, peers, keys or routes as part of this lab.
- Update affected `vm/docs/` and `vm/knowledge/okf/` alongside every meaningful implementation change or experiment. Separate planned behavior from observed evidence.
- Record consequential architecture changes in `vm/docs/adr/`; keep the architecture and track plans consistent.
- Never commit secrets or unredacted cluster credentials.
- Never infer permission to wipe a disk from its size or availability. Destructive operations need explicit targets, ownership checks and confirmation.
  The user has given standing approval for one case: `/dev/vdb` on iximiuz storage hosts node-01 to node-03 during a new deployment. Run the checks in `vm/docs/adr/0009-standing-disk-wipe-approval.md` and do not ask when they all pass. If any check fails, or for any other disk, host or provider, stop and ask.
- Pin deployment artifacts and record compatibility evidence before installing components. Validate stage prerequisites and document recovery.

- Keep documentation beginner friendly: use separate, copy-and-paste shell commands, explain their purpose and expected output, and identify which VM runs them. Use standard Ceph CLI commands for current lab verification; do not reintroduce custom validators or command wrappers without a concrete automation need.

- Write reusable VM documentation for any suitable Linux VM provider using ordinary SSH. Keep iximiuz/labctl, actual playground IDs, mesh addresses and observed deployment details in explicitly labeled provider guides, inventories or findings. Use clearly labeled example values and explain substitutions in main guides.

- The core VM setup requires three storage VMs, their second disks and the Ceph dashboard. Extra client VMs are optional extensions. Use labctl expose port for iximiuz UI access; document direct VM HTTPS access for VirtualBox and other providers.

- Current client extensions consume RBD from node-04 (native service) and node-05 (single-node k3s/Ceph-CSI). Distinguish Ceph storage HA from client/application HA, and explain shared raw versus replicated usable space. Keep newer Ceph credentials; do not relax authentication to accommodate older clients.

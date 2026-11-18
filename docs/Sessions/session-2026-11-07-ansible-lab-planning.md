---
name: session-2026-11-07-ansible-lab-planning
type: session
date: 2026-11-07
---

Planning for a new Ansible container to roll out updates to the Proxmox nodes and their guests, and to take over the Asscord playbooks. Build plan in [[homelab-update-automation]].

## Decided
- Semaphore, not Terraform ([[semaphore-for-update-automation]]).
- LXC 1200, hostname [[ansible-lab]], 10.0.30.x on [[VLAN30]], on [[pve-services]]. 2 cores, 2048 MB RAM, 512 MB swap, 20 GB disk.
- LXC updates via `pct exec` from the node; VM updates via `qm guest exec`.
- Web UI for Daniel only, behind Authentik forward auth. Manual and scheduled runs both to be explored.
- Heimdall in scope; Metasploitable2, DVWA, malware-win11, OPNsense and the VPSs out.
- ansible-lab gets a one-way tailnet exception to reach the Asscord staging VPS and dev-git.

## Found
- The vault's copy of the tailnet policy was two weeks stale. Replaced with the live one in [[tailscale-tailnet-policy]]; achilles, pve-ansible and the Asscord staging VPS exceptions recorded in [[tailscale-access-model]].
- [[pve-ansible]] is on the homelab tailnet and runs the Asscord deploys, not only Achilles work.
- How the Asscord deploy works, from the `ansible-asscord` README on pve-ansible: a hand-fetched clone of `[admin-user]/asscord` from [[dev-git]] at `~/asscord-src`, a pinned commit in the staging inventory, and the role packs that commit and pushes it to the VPS over SSH.
- `/home/ansible/asscord-vault-original.yml` on pve-ansible is an encrypted Ansible Vault copy outside the repo. Delete after the move.
- Claude Code sessions and chats outside this Project can't be searched from here, which is how the Asscord tailnet work went undocumented.

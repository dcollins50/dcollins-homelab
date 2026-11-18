---
name: ansible-lab
type: lxc
vmid: 1101
node: pve-services
ip: 10.0.30.x
vlan: VLAN30 (trust infra)
status: planned
role: Semaphore and Ansible for homelab updates and the Asscord deploys
---

Not built yet. VMID changed from 1200 to 1101 on Nov 16, 2026, because 1200 is now taken by pve-zap on [[pve-gateway]]. Planned Debian 13 LXC on [[pve-services]]: 2 cores, 2048 MB RAM, 512 MB swap, 20 GB disk on `local-lvm`. 10.0.30.x was unused per [[VLAN30]] on Nov 7, 2026, not checked live.

Runs Ansible through Semaphore ([[semaphore-for-update-automation]]). It will have root-level reach into every node and guest, so it is treated as Tier 0. Web UI for Daniel only, behind Authentik forward auth.

Build plan and open items: [[homelab-update-automation]].

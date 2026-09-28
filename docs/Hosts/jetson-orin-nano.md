---
name: jetson-orin-nano
type: physical-node
vmid: n/a
node: n/a
ip: 10.99.0.100
vlan: VLAN40
status: live
role: Local AI inference node
---

Racked alongside the Proxmox cluster. Runs local AI inference (Qwen 2.5 3B) independent of external APIs, for on-premises AI workloads and locally hosted model experimentation.

Planned Sep 23, 2026: move to [[VLAN11]] (AIops), see [[aiops-vlan]]. Planned address 10.0.11.10 (DHCP reservation). Stays on Tailscale after the move. Not moved yet, still on VLAN40.

No Wazuh agent yet (deferred — straightforward Debian-based install, not in scope currently).

## Related
- [[VLAN40]]
- [[VLAN11]]

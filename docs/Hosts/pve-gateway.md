---
name: pve-gateway
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.10
vlan: VLAN1 (mgmt), VLAN40/41 (lab VMs)
status: live
role: Security lab VMs, attack platform
---

HP EliteDesk G3. One of four Proxmox nodes, Corosync quorum member. Proxmox VE 9.1.1 / Kernel 6.17.2-1-pve.

## Hosted VMs
- [[kali-attack]] (300)
- [[metasploitable2]] (301)
- [[dvwa]] (302)
- [[malware-win11]] (400)

## Related
- [[VLAN1]]
- [[VLAN40]]
- [[VLAN41]]

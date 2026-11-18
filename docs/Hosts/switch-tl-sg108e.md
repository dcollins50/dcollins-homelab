---
name: switch-tl-sg108e
type: physical-node
vmid: n/a
node: n/a
ip: n/a
vlan: trunk (all VLANs)
status: live
role: VLAN tagging and trunking
---

TP-Link TL-SG108E managed switch. Handles VLAN tagging and trunking across all connected devices. Each port carries the appropriate PVID for its access VLAN; trunk ports carry tagged traffic to [[opnsense]] and the Proxmox nodes.

**VLANs configured:** 1, 10, 20, 30, 40, 41, 50, 51, 52, 55, 60. VLANs 53 and 54 are reserved for further DMZ segmentation.

## Port map
| Port | Connected to | Notes |
|------|--------------|-------|
| 1 | Proxmox node (pve-gateway or pve-env2, not yet confirmed) | Trunk |
| 2 | [[pve-services]] | Trunk. Also carries 51 and 52 |
| 3 | Proxmox node (pve-gateway or pve-env2, not yet confirmed) | Trunk |
| 4 | [[pve-env1]] | Trunk. Also carries 55 (CI) |
| 5 | [[opnsense]] LAN | Trunk to the firewall, carries every VLAN |
| 6 | [[jetson-orin-nano]] | Untagged access port |

## 802.1Q VLAN table (screenshot, Nov 16, 2026)
| VLAN | Name | Tagged | Untagged |
|------|------|--------|----------|
| 1 | Default | 5 | 1-4, 8 |
| 10 | SOC | 1-5 | |
| 20 | Services | 1-5 | |
| 30 | Dev | 1-5 | |
| 40 | Security1 | 1-5 | 6-7 |
| 41 | Security2 | 1-5 | |
| 50 | DMZ | 1-5 | |
| 51 | DMZBastion | 2, 5 | |
| 52 | DMZTunnels | 2, 5 | |
| 55 | CI | 4-5 | |
| 60 | Storage | 1-5 | |

Confirmed 2026-11-10: port 4 is pve-env1. Port 2 confirmed 2026-09-21. Ports 1 and 3 are still to be confirmed.

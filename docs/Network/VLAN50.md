---
name: VLAN50
vlan_id: 50
subnet: 10.0.50.0/24
status: live
role: DMZ1
---

**Firewall policy:** inbound 80/443 via Cloudflare Tunnel only. No route from DMZ to management, SOC, trust infrastructure, lab, or storage VLANs unless explicitly allowlisted.

## Hosts
| Host | IP |
|------|----|
| [[npm-dmz]] | 10.0.50.69 |
| [[ntfy]] | 10.0.50.69 (Docker container inside [[npm-dmz]]) |
| [[pve-authtunnel]] | 10.0.50.70 |
| Self-Hosted Website | planned |
| Stoat Messenger | planned |

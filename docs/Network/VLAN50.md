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
| [[npm-dmz]] | 10.0.50.x |
| [[ntfy]] | 10.0.50.x (Docker container inside [[npm-dmz]]) |
| [[pve-authtunnel]] | 10.0.50.x |
| [[dev-git]] | 10.0.50.x |
| [[nimbus]] | 10.0.50.x |
| Self-Hosted Website | planned |
| Stoat Messenger | planned |

---
name: VLAN40
vlan_id: 40
subnet: 10.99.0.0/24
status: live
role: Security Lab 1
---

Flat network — all VLAN40 hosts reach each other directly, no additional firewall rule required.

**Firewall policy:** restricted outbound internet for tool updates only. No route to production, services, trust infrastructure, SOC, or management VLANs.

## Hosts
| Host | IP |
|------|----|
| [[kali-attack]] (VM 300) | 10.99.0.x |
| [[metasploitable2]] (VM 301) | 10.99.0.x |
| [[dvwa]] (VM 302) | 10.99.0.x |
| [[jetson-orin-nano]] | 10.99.0.100 |

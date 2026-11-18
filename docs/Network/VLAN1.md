---
name: VLAN1
vlan_id: 1
subnet: 10.0.0.0/24
status: live
role: Management
---

Carries Proxmox node management traffic and OPNSense management access. Access restricted to administrator IPs only. No unsolicited inbound from other VLANs. Root SSH login disabled on all four Proxmox nodes — access requires a non-root account.

## Hosts
| Host | IP |
|------|----|
| [[opnsense]] (mgmt) | 10.0.0.x |
| [[pve-gateway]] | 10.0.0.x |
| [[pve-services]] | 10.0.0.x |
| [[pve-env1]] | 10.0.0.x |
| [[pve-env2]] | 10.0.0.x |

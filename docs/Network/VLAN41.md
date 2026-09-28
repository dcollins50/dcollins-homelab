---
name: VLAN41
vlan_id: 41
subnet: 10.99.1.0/24
status: live
role: Security Lab 2
---

**Firewall policy:** fully air-gapped. No internet access, no route to or from any other VLAN — including [[VLAN40]], ruling it out as a lateral-movement target.

## Hosts
| Host | IP |
|------|----|
| [[malware-win11]] (VM 400) | 10.99.1.x |

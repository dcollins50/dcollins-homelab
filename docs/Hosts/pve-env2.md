---
name: pve-env2
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.13
vlan: VLAN1 (mgmt), VLAN10 (SOC)
status: live
role: SOC stack — dedicated to ELK and Wazuh
---

HP EliteDesk G6. One of four Proxmox nodes, Corosync quorum member. 32GB RAM total.

Reserved exclusively for the core SOC stack (Elastic, Wazuh). Remaining capacity is reserved for additional agents, not new services — that's why soar-host and pve-iris run on [[pve-env1]] instead.

## Hosted VMs
- [[soc-stack-vm|soc-stack]] (VM 600, 10.0.10.10) — Elasticsearch, Logstash, Kibana
- [[wazuh-manager]] (VM 601, 10.0.10.11) — Wazuh SIEM Manager

## Related
- [[VLAN1]]
- [[VLAN10]]

---
name: SOC-Stack
type: moc
status: live
---

Detection, log analysis, alert automation, and incident case management. Isolated on [[VLAN10]], no unsolicited inbound from other VLANs, everything over TLS via the internal PKI. This is a live working stack, not a demo — agents deployed across the cluster, logs ingesting in real time, dashboards populated with live data.

Host detail: [[soc-stack-vm|soc-stack]] (ELK), [[wazuh-manager]], [[soar-host]] (Shuffle SOAR), [[pve-iris]] (DFIR-IRIS). Buildout history/phase docs: see the Projects/soc-stack-buildout/ folder.

## Components
| Component | Host | Role |
|-----------|------|------|
| ELK Stack | [[soc-stack-vm]] | Elasticsearch, Logstash, Kibana |
| Wazuh Manager | [[wazuh-manager]] | SIEM manager |
| Shuffle SOAR | [[soar-host]] | Alert automation |
| DFIR-IRIS | [[pve-iris]] | Incident case tracking |

Addresses, VM IDs and host nodes are on the host notes and in [[VLAN10]].

pve-env2 is reserved exclusively for the core ELK/Wazuh stack; soar-host and pve-iris run on pve-env1 since pve-env2's remaining capacity is reserved for additional agents.

## Firewall
The SOC network's firewall policy and log flow are documented in [[VLAN10]]. Open item: [[soar-host]] and [[pve-iris]] still have no dedicated rules beyond the general SOC policy.

## Pending items
| Item | Status |
|------|--------|
| Standard-PC-Q35-ICH9-2009 hostname | Noisy hostname needs `hostnamectl` fix |
| Suricata IPS mode | Currently detection-only (IDS) |
| soar-host / pve-iris dedicated firewall rules | Not yet scoped, rely on general SOC net policy |

## Related
- [[Infrastructure]]
- [[Network]]
- [[PKI]]

## SOPs
- [[sop-soc-alert-pipeline|SOC Alert Pipeline]]
- [[sop-log-source-onboarding|Onboard a New Log Source]]

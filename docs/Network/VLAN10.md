---
name: VLAN10
vlan_id: 10
subnet: 10.0.10.0/24
status: live
role: SOC
---

Hosts detection/log analysis (ELK, Wazuh), alert automation (Shuffle SOAR), incident case management (DFIR-IRIS).

**Firewall policy:** general outbound HTTP/HTTPS (80/443). DNS restricted to OPNSense as sole resolver, all other DNS destinations blocked. LAN SSH permitted to [[wazuh-manager]] only (TCP 22); wazuh-manager itself permitted agentless SSH back to OPNSense. Agents reach wazuh-manager on TCP/UDP 1514-1515. [[soar-host]] and [[pve-iris]] don't yet have dedicated rules beyond the general SOC net outbound policy. soc-stack has a dedicated WireGuard rule to [[heimdall]].

**Log flow:** OPNSense Suricata EVE JSON → Logstash port 5144 → Elasticsearch. Wazuh agent alerts → Filebeat → Elasticsearch. Heimdall rsyslog → Logstash port 5146 → Elasticsearch.

## Hosts
| Host | IP |
|------|----|
| [[soc-stack-vm]] (VM 600) | 10.0.10.10 |
| [[wazuh-manager]] (VM 601) | 10.0.10.11 |
| [[soar-host]] (VM 602) | 10.0.10.12 |
| [[pve-iris]] (VM 604) | 10.0.10.13 |

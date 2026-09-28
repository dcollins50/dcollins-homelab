---
name: wazuh-manager
type: vm
vmid: 601
node: pve-env2
ip: 10.0.10.11
vlan: VLAN10
status: live
role: Wazuh SIEM Manager
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

**Specs:** 8GB RAM, 2 cores.
**Version:** Wazuh 4.14.4.

TLS via `sslmanager.cert`/`sslmanager.key` for agent enrollment (port 1515, `wazuh-authd`). Filebeat ships alerts to [[soc-stack-vm|soc-stack]] over HTTPS.

**Agents deployed on:** [[services-host]] (200), [[soc-stack-vm|soc-stack]] (600), wazuh-manager itself (601), [[ubuntu-401]] (401), [[kali-attack]] (300), and all four Proxmox nodes ([[pve-gateway]], [[pve-services]], [[pve-env1]], [[pve-env2]]).

**No agent on:** [[metasploitable2]], [[dvwa]], [[malware-win11]] (intentionally compromised/isolated).

OPNSense integration: syslog forwarding, plus an agentless SSH check against the OPNSense management interface.

**Past incident:** hit 100% disk full Sep 13, 2026 from a 62G unrotated `alerts.json`/`alerts.log` that never rotated when the cluster went down around Aug 20. Fixed via truncate, freed disk back to 57G available.

**Firewall:** LAN SSH permitted to this host only (TCP 22) for agent management; this host is permitted agentless SSH back to OPNSense. Agents on VLAN10 reach it on TCP/UDP 1514-1515.

## Related
- [[pve-env2]]
- [[soc-stack-vm]]
- [[VLAN10]]

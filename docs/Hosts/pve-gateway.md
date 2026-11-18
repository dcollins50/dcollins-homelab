---
name: pve-gateway
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.x
vlan: VLAN1 (mgmt), VLAN40/41 (lab VMs)
status: live
role: Security lab VMs, attack platform
---

HP EliteDesk G3. One of four Proxmox nodes, Corosync quorum member. Proxmox VE 9.2.26 / kernel 7.0.14-28-pve since Nov 8, 2026 (was 9.1.1 / 6.17.2-1-pve). Kernels 6.17.2-1-pve and 6.17.13-21-pve are still installed as fallbacks; none pinned.

**Package repositories:** Proxmox No-Subscription on `trixie`, added Nov 8, 2026. Before that the only Proxmox repository was the Proxmox VE 8 (`bookworm`) No-Subscription entry in `pve-no-subscription.list`, left over from the 8 to 9 upgrade, so the node got no Proxmox updates. That entry is disabled, not deleted. Elastic and Wazuh repositories use suite `stable`, which Proxmox flags as misconfigured; harmless. `wazuh-agent` and `filebeat` are on `apt-mark hold` until the SOC update. See [[homelab-update-automation]].

## Hosted VMs
- [[kali-attack]] (300)
- [[metasploitable2]] (301)
- [[dvwa]] (302)
- [[malware-win11]] (400)
- [[pve-zap]] (LXC 1200, 10.99.0.x) OWASP ZAP scanner

## Related
- [[VLAN1]]
- [[VLAN40]]
- [[VLAN41]]

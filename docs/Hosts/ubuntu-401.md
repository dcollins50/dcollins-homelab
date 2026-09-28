---
name: ubuntu-401
type: vm
vmid: 401
node: pve-services
ip: 10.0.30.40
vlan: VLAN30
status: live
role: General services
---

**Login:** `admin`, password auth (SSH key not yet copied as of Sep 25, 2026).

General-purpose Ubuntu VM on the Trust Infrastructure VLAN. Runs a Wazuh agent.

IP 10.0.30.40/24, confirmed in [[incident-2026-01-24-vlan-connectivity-fixes]] and [[soc-phase2.5-midweek-state]], and by Daniel on Sep 24, 2026.

## Related
- [[pve-services]]
- [[VLAN30]]

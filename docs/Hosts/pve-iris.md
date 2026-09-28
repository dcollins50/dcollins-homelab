---
name: pve-iris
type: vm
vmid: 604
node: pve-env1
ip: 10.0.10.13
vlan: VLAN10
status: live
role: DFIR-IRIS — incident case tracking
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Full clone of [[docker-host-template]] (VM 603). Sized 4 vCPU/8GB RAM. IP corrected to 10.0.10.13/24 after discovering the template's clone still carried soar-host's old IP (10.0.10.12), which would have conflicted with [[soar-host]] (VM 602).

Receives tickets from [[soar-host]]'s Shuffle SOAR debounce workflow only if a Uptime Kuma DOWN event persists past the recheck.

Does not yet have a dedicated firewall rule beyond the general SOC net outbound policy.

Deployed here because [[pve-env2]] (the dedicated SOC stack node) is capped out, with remaining capacity reserved for additional Wazuh agents rather than new services.

## Related
- [[pve-env1]]
- [[soar-host]]
- [[VLAN10]]

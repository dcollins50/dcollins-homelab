---
name: soar-host
type: vm
vmid: 602
node: pve-env1
ip: 10.0.10.12
vlan: VLAN10
status: live
role: Shuffle SOAR — alert automation
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Deployed Sep 19, 2026. Docker installed, repo cloned, OpenSearch prerequisites completed. All six containers running: frontend, backend, worker, orborus, security, opensearch.

LAN firewall rule added (Workstation → soar_host alias, ports 3001/3443) to reach the web UI.

**Debounce workflow:** waits 60 seconds after Uptime Kuma's confirmed-DOWN event (post Kuma's own retries), then rechecks; only creates an [[pve-iris]] ticket if still down. Goal is reducing false-positive ticket generation.

Does not yet have a dedicated firewall rule beyond the general SOC net outbound policy.

Runs on [[pve-env1]] rather than [[pve-env2]] because pve-env2's remaining capacity is reserved for additional Wazuh agents, not new services.

## Related
- [[pve-env1]]
- [[pve-iris]]
- [[VLAN10]]

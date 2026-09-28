---
name: services-host
type: vm
vmid: 200
node: pve-env1
ip: 10.0.20.30
vlan: VLAN20
status: live
role: Docker Host 1
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Running: Vaultwarden, internal NPM (Nginx Proxy Manager, separate from npm-dmz), Gitea, Stremio (planned removal — Stremio now runs from the workstation directly instead), Portainer, Uptime Kuma.

Uptime Kuma here monitors TLS certificate validity for all internal HTTPS endpoints and alerts on approaching expiry.

Wazuh Docker wodle enabled; the `wazuh` user is in the `docker` group so container events can be collected without running the agent as root.

## Related
- [[pve-env1]]
- [[VLAN20]]

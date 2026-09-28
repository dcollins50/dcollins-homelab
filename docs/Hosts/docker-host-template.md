---
name: docker-host-template
type: vm
vmid: 603
node: pve-env1
ip: n/a
vlan: n/a
status: template
role: Template for cloning new Docker-host VMs
---

General-purpose Docker VM template, not tied to services-host. Cloned to create [[pve-iris]] (VMID 604).

Note: the template's IP still carries an old soar-host value (10.0.10.12) — when cloning, correct the IP before bringing the clone online, or it will conflict with [[soar-host]].

## Related
- [[pve-env1]]
- [[pve-iris]]

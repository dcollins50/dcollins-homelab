---
name: metasploitable2
type: vm
vmid: 301
node: pve-gateway
ip: 10.99.0.x
vlan: VLAN40
status: live
role: Intentionally vulnerable Linux target
---

**Login:** `msfadmin`, password auth (default credentials, intentional — never copy an SSH key here).

Deliberately vulnerable Linux distribution for penetration testing practice. Exposes weak SSH credentials, vulnerable FTP, unpatched web apps, misconfigured services. Reachable directly from [[kali-attack]] on the same flat VLAN40.

No Wazuh agent (intentionally compromised state).

## Related
- [[pve-gateway]]
- [[VLAN40]]

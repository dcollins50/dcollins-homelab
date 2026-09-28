---
name: dvwa
type: vm
vmid: 302
node: pve-gateway
ip: 10.99.0.x
vlan: VLAN40
status: live
role: Vulnerable web application target
---

**Login:** `sploitme`, password auth (intentional — never copy an SSH key here).

Damn Vulnerable Web Application. Practice environment for SQL injection, XSS, command injection, file inclusion, CSRF. Security level is configurable for progressive difficulty. Reachable directly from [[kali-attack]] on the same flat VLAN40; Burp Suite is the primary proxy used against it.

No Wazuh agent (intentionally compromised state).

## Related
- [[pve-gateway]]
- [[VLAN40]]

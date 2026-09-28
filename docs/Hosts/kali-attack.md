---
name: kali-attack
type: vm
vmid: 300
node: pve-gateway
ip: 10.99.0.10
vlan: VLAN40
status: live
role: Primary attack platform
---

**Login:** `pistol`, password auth (SSH key not yet copied as of Sep 25, 2026).

Primary penetration testing platform. Offensive tooling, network reconnaissance, exploitation, post-exploitation practice. Restricted outbound internet for tool/wordlist updates only.

Tools in regular use: Nmap, Metasploit Framework, Burp Suite, Gobuster, Hydra, John the Ripper, Impacket.

Sits flat on VLAN40 with [[metasploitable2]] and [[dvwa]] — reaches both directly, no additional routing.

Runs a Wazuh agent (monitors offensive activity, telemetry flows to [[soc-stack-vm|soc-stack]]), giving a bidirectional view: attack traffic visible in OPNSense/Suricata, system-level activity on the attacker box visible via Wazuh.

## Related
- [[pve-gateway]]
- [[VLAN40]]

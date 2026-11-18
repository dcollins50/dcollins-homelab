---
name: kali-attack
type: vm
vmid: 300
node: pve-gateway
ip: 10.99.0.x
vlan: VLAN40
status: live
role: Primary attack platform
---

**Login:** `pistol`, password auth (SSH key not yet copied as of Sep 25, 2026).

Primary penetration testing platform. Offensive tooling, network reconnaissance, exploitation, post-exploitation practice. Restricted outbound internet for tool/wordlist updates only.

Tools in regular use: Nmap, Metasploit Framework, Burp Suite, Gobuster, Hydra, John the Ripper, Impacket.

Sits flat on VLAN40 with [[metasploitable2]] and [[dvwa]] — reaches both directly, no additional routing.

Runs a Wazuh agent (monitors offensive activity, telemetry flows to [[soc-stack-vm|soc-stack]]), giving a bidirectional view: attack traffic visible in OPNSense/Suricata, system-level activity on the attacker box visible via Wazuh.

## Update Oct 31, 2026: DNS exempt from Pi-hole blocking
DNS is `1.1.1.1` and `1.0.0.1` directly (NetworkManager profile `Wired connection 1`, `ipv4.ignore-auto-dns yes`), so Kali is not filtered by Pi-hole. This is the only lab host with that exemption, by Daniel's decision. A VLAN40 firewall rule allows `10.99.0.x` to `cloudflare_dns` on port 53, above the DNS block. Rollback and details in [[dns-phase1-consolidation]]. Before today Kali used `10.99.0.x` (Unbound).

## Related
- [[pve-gateway]]
- [[VLAN40]]

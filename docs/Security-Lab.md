---
name: Security-Lab
type: moc
status: live
---

Network-isolated environment for penetration testing practice, adversary simulation, and hands-on offensive/defensive work. Host detail: [[kali-attack]], [[metasploitable2]], [[dvwa]], [[malware-win11]], [[jetson-orin-nano]]. All lab VMs run on [[pve-gateway]].

## Network isolation
Two VLANs, enforced at OPNSense. Neither has any route to production, services, trust infrastructure, SOC, management, or storage VLANs — default-deny, no exceptions.

| VLAN | Hosts | Internet | Lateral access |
|------|-------|----------|-----------------|
| [[VLAN40]] | Kali, Metasploitable2, DVWA, Jetson | Restricted (tool updates only) | Flat — all VLAN40 hosts reach each other directly |
| [[VLAN41]] | malware-win11 | None | Fully air-gapped, no route anywhere including VLAN40 |

## Lab use cases
- **Network penetration testing** — Kali against Metasploitable2: service enumeration, vulnerability ID, exploitation across a range of CVEs/misconfigurations
- **Web application testing** — Kali against DVWA: OWASP Top 10, Burp Suite as the primary intercepting proxy
- **Malware analysis** — executing/observing samples in the Windows 11 sandbox; air-gapped network prevents outbound comms while still allowing behavior observation
- **AI-assisted analysis** — Jetson Orin Nano running local inference for on-prem AI assistance without external API calls

## Wazuh agent coverage
Agent deployed on Kali only. Gives a bidirectional view: attack traffic visible in OPNSense/Suricata logs, system-level activity on the attacker box visible via the Wazuh agent. Vulnerable targets (Metasploitable2, DVWA) and the sandbox (malware-win11) don't run agents given their intentionally compromised/isolated state.

## Planned work
| Item | Description |
|------|-------------|
| Active Directory lab | Windows Server 2022 DC on pve-services, purely an attack-range target for AD lab scenarios. Kali and malware-win11 selectively domain-joined for practice. Not related to production identity (Authentik). Network+ is done, so no longer blocked. Not started. |
| Wazuh agent on Jetson | Deferred — straightforward Debian-based install, not yet in scope |

## External redirector
[[vps-redirector]] is a public-facing the provider VPS that fronts the lab for engagements: targets hit the VPS's public IP, which relays back to [[kali-attack]] on [[VLAN40]], keeping the home IP hidden and containing a burned VPS. The relay currently runs over Bore (cleartext, terminates on Kali); the intended replacement is a WireGuard tunnel terminating on [[opnsense]], scoped so the tunnel can reach only Kali's forwarded ports. Offensive methodology lives in the pentest-notes repo; the box as infrastructure and its hardening live in [[vps-redirector]] and [[sop-harden-debian-vps]].

## Related
- [[Infrastructure]]
- [[Network]]
- [[SOC-Stack]]

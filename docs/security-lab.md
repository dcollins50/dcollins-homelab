# Security Lab

This document covers the security lab environment, including network isolation, hosted targets, tooling, and the AI inference node. The lab is used for penetration testing practice, adversary simulation, and hands-on offensive and defensive security work.

---

## Network Isolation

The security lab runs across two VLANs enforced at the OPNSense firewall layer. Neither VLAN has any route to production, services, trust infrastructure, SOC, management, or storage VLANs. This isolation is enforced by default-deny firewall rules with no exceptions.

| VLAN | Hosts | Internet Access | Lateral Access |
|------|-------|----------------|----------------|
| VLAN40 | Kali Linux VM, Metasploitable2, DVWA, Jetson Orin Nano | Restricted (tool updates only) | Flat network — all VLAN40 hosts can reach each other directly |
| VLAN41 | malware-win11 | None | Fully air-gapped — no route to or from any other VLAN, including VLAN40 |

Kali, Metasploitable2, and DVWA all sit on the same broadcast domain (VLAN40), so Kali can attack them directly with no additional routing required. VLAN41 is a separate, fully isolated pocket reserved for the malware analysis sandbox — it has no connectivity anywhere, including to Kali, which rules out using it as a lateral-movement target from VLAN40.

---

## Physical Hardware

### Jetson Orin Nano

| Field | Value |
|-------|-------|
| Role | Local AI inference node |
| Network | VLAN40, 10.99.0.100 |
| Model loaded | Qwen 2.5 3B |
| Physical | Racked alongside the Proxmox cluster |

The Jetson runs local AI inference independent of external APIs. It is used for on-premises AI workloads and experimentation with locally hosted language models.

---

## Virtual Machines

All lab VMs run on pve-gateway (HP EliteDesk G3, 10.0.0.10).

### Kali Linux (VM 300)

| Field | Value |
|-------|-------|
| VM ID | 300 |
| Network | VLAN40 |
| Role | Primary attack platform |

The primary penetration testing platform. Used for offensive tooling, network reconnaissance, exploitation, and post-exploitation practice. Has restricted outbound internet access for tool and wordlist updates.

Tools in regular use include Nmap, Metasploit Framework, Burp Suite, Gobuster, Hydra, John the Ripper, and Impacket.

---

### Metasploitable2 (VM 301)

| Field | Value |
|-------|-------|
| VM ID | 301 |
| Network | VLAN40 |
| Role | Intentionally vulnerable Linux target |

Metasploitable2 is a deliberately vulnerable Linux distribution designed for penetration testing practice. It exposes a range of exploitable services including weak SSH credentials, vulnerable FTP, unpatched web applications, and misconfigured network services. Reachable directly from Kali on the same flat VLAN40 network.

---

### DVWA — Damn Vulnerable Web Application (VM 302)

| Field | Value |
|-------|-------|
| VM ID | 302 |
| Network | VLAN40 |
| Role | Vulnerable web application target |

DVWA provides a web application environment for practicing common web vulnerabilities including SQL injection, cross-site scripting, command injection, file inclusion, and CSRF. Security level is configurable for progressive difficulty. Reachable directly from Kali on the same flat VLAN40 network.

---

### Windows 11 Malware Sandbox (VM 400)

| Field | Value |
|-------|-------|
| VM ID | 400 |
| Network | VLAN41 (air-gapped) |
| Role | Malware analysis and Windows attack simulation |

A Windows 11 VM used for malware analysis and Windows-specific attack scenarios. Complete network isolation, no route to or from any other VLAN including VLAN40, prevents any malware executed in this environment from reaching external infrastructure or internal networks. Also used for practicing Windows privilege escalation and credential attacks in preparation for Active Directory lab work.

---

## Lab Use Cases

**Network penetration testing:** Using Kali against Metasploitable2 to practice network service enumeration, vulnerability identification, and exploitation across a range of CVEs and misconfigurations.

**Web application testing:** Using Kali against DVWA to practice OWASP Top 10 vulnerabilities in a controlled environment. Burp Suite is the primary proxy for intercepting and manipulating HTTP traffic.

**Malware analysis:** Executing and observing malware samples in the Windows 11 sandbox. The air-gapped network prevents any outbound communication while still allowing observation of system behavior.

**AI-assisted analysis:** The Jetson Orin Nano runs local inference for on-premises AI assistance without sending data to external APIs.

---

## Wazuh Agent Coverage

Wazuh agents are deployed on Kali Linux (VM 300) to monitor offensive activity and generate telemetry that flows back to the SOC stack. This creates a bidirectional view — attack traffic is visible in the OPNSense firewall logs and Suricata alerts in Kibana, and system-level activity on the attacker machine is visible through the Wazuh agent.

Vulnerable targets on VLAN40 (Metasploitable2, DVWA) and the sandbox on VLAN41 (malware-win11) do not run Wazuh agents given their intentionally compromised or isolated state.

---

## Planned Work

| Item | Description |
|------|-------------|
| Active Directory lab | Windows Server 2022 domain controller on pve-services, used purely as an attack-range target for AD lab scenarios. Kali and malware-win11 selectively domain-joined for practice. Not related to production identity, which runs on Authentik. Deferred until after Network+ exam. |
| Wazuh agent on Jetson | Deferred — straightforward Debian-based install, not yet in scope |

---

## Related Documentation

- [Network Architecture](network.md)
- [Infrastructure](infrastructure.md)
- [SOC Stack](soc-stack.md)

# Security Lab

This document covers the security lab environment, including network isolation, hosted targets, tooling, and the AI inference node. The lab is used for penetration testing practice, adversary simulation, and hands-on offensive and defensive security work.

---

## Network Isolation

The security lab runs across two isolated VLANs enforced at the OPNSense firewall layer. Neither VLAN has any route to production, services, SOC, management, or storage VLANs. This isolation is enforced by default-deny firewall rules with no exceptions.

| VLAN | Hosts | Internet Access | Lateral Access |
|------|-------|----------------|----------------|
| VLAN40 | Kali Linux VM, Jetson Orin Nano | Restricted (tool updates only) | None |
| VLAN41 | Metasploitable2, DVWA, malware-win11 | None | None |

VLAN41 is fully air-gapped. Hosts in VLAN41 cannot reach the internet or any other network segment. This ensures malware samples and vulnerable targets cannot be used as pivot points or exfiltrate data.

Kali in VLAN40 can reach targets in VLAN41, simulating an attacker on a compromised internal host attempting lateral movement.

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

All lab VMs run on pve-lab (HP EliteDesk G3, 10.0.0.10).

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
| Network | VLAN41 (air-gapped) |
| Role | Intentionally vulnerable Linux target |

Metasploitable2 is a deliberately vulnerable Linux distribution designed for penetration testing practice. It exposes a range of exploitable services including weak SSH credentials, vulnerable FTP, unpatched web applications, and misconfigured network services. No internet access. No route to other VLANs.

---

### DVWA — Damn Vulnerable Web Application (VM 302)

| Field | Value |
|-------|-------|
| VM ID | 302 |
| Network | VLAN41 (air-gapped) |
| Role | Vulnerable web application target |

DVWA provides a web application environment for practicing common web vulnerabilities including SQL injection, cross-site scripting, command injection, file inclusion, and CSRF. Security level is configurable for progressive difficulty. No internet access. No route to other VLANs.

---

### Windows 11 Malware Sandbox (VM 400)

| Field | Value |
|-------|-------|
| VM ID | 400 |
| Network | VLAN41 (air-gapped) |
| Role | Malware analysis and Windows attack simulation |

A Windows 11 VM used for malware analysis and Windows-specific attack scenarios. Complete network isolation prevents any malware executed in this environment from reaching external infrastructure or internal networks. Also used for practicing Windows privilege escalation and credential attacks in preparation for Active Directory lab work.

---

## Lab Use Cases

**Network penetration testing:** Using Kali against Metasploitable2 to practice network service enumeration, vulnerability identification, and exploitation across a range of CVEs and misconfigurations.

**Web application testing:** Using Kali against DVWA to practice OWASP Top 10 vulnerabilities in a controlled environment. Burp Suite is the primary proxy for intercepting and manipulating HTTP traffic.

**Malware analysis:** Executing and observing malware samples in the Windows 11 sandbox. The air-gapped network prevents any outbound communication while still allowing observation of system behavior.

**AI-assisted analysis:** The Jetson Orin Nano runs local inference for on-premises AI assistance without sending data to external APIs.

---

## Wazuh Agent Coverage

Wazuh agents are deployed on Kali Linux (VM 300) to monitor offensive activity and generate telemetry that flows back to the SOC stack. This creates a bidirectional view — attack traffic is visible in the OPNSense firewall logs and Suricata alerts in Kibana, and system-level activity on the attacker machine is visible through the Wazuh agent.

Vulnerable targets in VLAN41 do not run Wazuh agents given their intentionally compromised state.

---

## Planned Work

| Item | Description |
|------|-------------|
| Active Directory lab | Windows Server 2022 domain controller on pve-services. Kali and malware-win11 selectively domain-joined for AD attack scenarios. Deferred until after Network+ exam. |
| Wazuh agent on Jetson | Deferred — straightforward Debian-based install, not yet in scope |

---

## Related Documentation

- [Network Architecture](network.md)
- [Infrastructure](infrastructure.md)
- [SOC Stack](soc-stack.md)

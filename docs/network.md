# Network Architecture

This document covers the full network topology, VLAN design, firewall architecture, and traffic flow for the homelab environment. The network is built around a default-deny security posture with explicit allowlists governing all inter-VLAN communication.

---

## Physical Topology

```
Internet
    |
Heimdall (RPi5) — WiFi Bridge / WAN Gateway
    |
OPNSense Firewall (Dedicated HP EliteDesk G3)
    |
TP-Link TL-SG108E (Managed Switch)
    |
    +— VLAN1  (Management)
    +— VLAN10 (SOC)
    +— VLAN20 (Trusted LAN)
    +— VLAN30 (Services)
    +— VLAN40 (Security Lab)
    +— VLAN41 (Isolated Lab)
    +— VLAN50 (DMZ) [Planned]
    +— VLAN51 (SSH Bastion) [Planned]
    +— VLAN60 (Storage)
```

---

## WAN Path

Internet connectivity enters through Heimdall, a Raspberry Pi 5 racked alongside the cluster. Heimdall bridges the WiFi uplink (`wlan0`) to its wired interface (`eth0`), which connects to the WAN port of the dedicated OPNSense firewall. Heimdall also runs Pi-hole for DNS filtering and WireGuard as a VPN gateway for remote administration.

Heimdall has a real public IPv4 address with no CGNAT, which enables direct WireGuard connectivity without relay.

---

## OPNSense Firewall

OPNSense runs on a dedicated HP EliteDesk G3 that is not part of the Proxmox cluster. It handles all routing, NAT, VLAN enforcement, and intrusion detection. No traffic passes between VLANs without an explicit OPNSense firewall rule permitting it.

### Suricata IDS/IPS

Suricata runs on OPNSense in detection-only mode (IDS) on the WAN interface. EVE JSON logs are forwarded to the ELK stack via Logstash for ingestion into Kibana. Promotion to inline blocking mode (IPS) is a pending hardening item.

### Firewall Design Principles

- Default-deny on all VLAN interfaces
- Explicit allowlists for all permitted inter-VLAN traffic
- Aliases used for common host groups (Wazuh_Manager, Wazuh_Agents)
- Anti-spoofing and RFC1918 blocking on WAN interface
- No inbound port forwarding rules on WAN (all external access via WireGuard or planned Cloudflare Tunnel)

---

## Managed Switch

**Model:** TP-Link TL-SG108E

The switch handles VLAN tagging and trunking across all connected devices. Each port is configured with the appropriate PVID for its access VLAN, with trunk ports carrying tagged traffic to the OPNSense firewall and Proxmox nodes.

**VLANs configured on switch:** 1, 10, 20, 30, 40, 41, 50, 51, 60

---

## VLAN Design

### VLAN1 — Management

The management VLAN carries Proxmox node management traffic and OPNSense management access. All four Proxmox nodes and OPNSense are reachable on this VLAN. Access is restricted to administrator IPs only.

| Host | IP Range |
|------|----------|
| OPNSense (management) | 10.0.0.1 |
| pve-lab | 10.0.0.10 |
| pve-services | 10.0.0.11 |
| pve-env1 | 10.0.0.12 |
| pve-SOC | 10.0.0.13 |

**Firewall policy:** Management VLAN access restricted to administrator subnet. No unsolicited inbound from other VLANs.

---

### VLAN10 — SOC

Dedicated to the SOC stack. Contains the ELK Stack VM and Wazuh Manager VM only. No other services run in this VLAN.

| Host | IP |
|------|----|
| soc-stack (VM 600) | 10.0.10.10 |
| wazuh-manager (VM 601) | 10.0.10.11 |

**Firewall policy:** Agents on other VLANs may reach wazuh-manager on TCP/UDP 1514-1515 for log shipping. SOC DNS permitted TCP/UDP 53. LAN SSH to wazuh-manager permitted on TCP 22 for agent management. No unsolicited inbound from other VLANs.

**Log flow:** OPNSense Suricata EVE JSON → Logstash (port 5144) → Elasticsearch. Wazuh agent alerts → Filebeat → Elasticsearch.

---

### VLAN20/30 — Trusted LAN / Services

Production self-hosted services run in VLAN20 and VLAN30. Docker workloads run on pve-env1 (VLAN20). The internal PKI and supporting VMs run on pve-services (VLAN30).

| Host | IP | VLAN |
|------|----|------|
| services-host (VM 200) | 10.0.20.30 | VLAN20 |
| kalshi-mm (VM 290) | 10.0.20.50 | VLAN20 |
| services-host2 (VM 700) | 10.0.20.31 | VLAN20 |
| ubuntu (VM 401) | 10.0.30.x | VLAN30 |
| pve-ca-root (VM 500) | 10.0.30.x | VLAN30 |
| pve-ca-intermediate (VM 501) | 10.0.30.x | VLAN30 |

**Firewall policy:** Outbound internet permitted. No unsolicited inbound from other VLANs. VLAN20 and VLAN30 are isolated from each other except for explicit allowlisted traffic.

---

### VLAN40/41 — Security Lab

The security lab is fully isolated from all production VLANs. VLAN40 hosts the Kali Linux attack VM and the Jetson Orin Nano. VLAN41 hosts vulnerable target VMs (Metasploitable2, DVWA, malware-win11).

| Host | IP | VLAN |
|------|----|------|
| kali-attack (VM 300) | 10.99.0.x | VLAN40 |
| Jetson Orin Nano | 10.99.0.100 | VLAN40 |
| metasploitable2 (VM 301) | 10.99.1.x | VLAN41 |
| dvwa (VM 302) | 10.99.1.x | VLAN41 |
| malware-win11 (VM 400) | 10.99.1.x | VLAN41 |

**Firewall policy:** VLAN41 is fully air-gapped — no internet access and no route to any other VLAN. VLAN40 has restricted outbound internet for tool updates only. No route from either lab VLAN to production, services, SOC, or management VLANs.

---

### VLAN50 — DMZ (Planned)

The DMZ will host all public-facing services. Traffic will enter exclusively via Cloudflare Tunnel — no inbound port forwarding will be configured on the WAN interface. Nginx Proxy Manager will handle reverse proxying from the tunnel to internal services.

**Planned services:** Self-Hosted Website, Stoat Messenger, Nginx Proxy Manager

**Planned firewall policy:** Inbound 80/443 via Cloudflare Tunnel only. Explicit allowlist for DMZ to internal services where required. No route from DMZ to management, SOC, lab, or storage VLANs.

---

### VLAN51 — SSH Bastion (Planned)

A dedicated hardened VM will serve as the single SSH entry point for all internal nodes. Direct SSH from workstations to internal nodes will be blocked at the firewall once the bastion is live. All SSH access will route through the bastion via ProxyJump.

**Planned firewall policy:** Bastion IP is the only permitted SSH source to all internal VLANs. Workstation-to-internal SSH blocked.

---

### VLAN60 — Storage

The storage VLAN carries NAS traffic. No internet access. Management access only from the management VLAN.

---

## DNS Architecture

| Layer | Role |
|-------|------|
| Pi-hole (Heimdall) | Network-wide DNS filtering and ad blocking, primary DNS for all VLANs |
| OPNSense Unbound | Query forwarding — `homelab.local` queries forwarded to internal DNS at 192.168.100.1 |
| Internal DNS records | `elasticsearch.homelab.local` → NPM IP |

All VMs use Pi-hole as their DNS resolver. The Pi-hole is at 192.168.100.1 on the Heimdall RPi5.

---

## Remote Access

Remote administration is handled entirely via WireGuard VPN hosted on Heimdall. No management interfaces are exposed directly to the internet. The WireGuard endpoint is reachable via Heimdall's real public IPv4.

Cloudflare Tunnels are planned for public-facing web services only and will not carry management traffic.

---

## Traffic Flow Reference

| Flow | Path | Status |
|------|------|--------|
| Internet → internal services | WireGuard VPN only | Live |
| Internet → public services | Cloudflare Tunnel → NPM → DMZ | Planned |
| Wazuh agents → manager | TCP/UDP 1514-1515, all VLANs → VLAN10 | Live |
| OPNSense logs → ELK | Syslog → Logstash port 5144 → Elasticsearch | Live |
| Suricata EVE → ELK | EVE JSON → Logstash → Elasticsearch | Live |
| Internal services → PKI | VLAN20/30 → VLAN30 CA VMs | Live |
| Admin workstation → nodes | WireGuard → VLAN1 management | Live |
| Admin workstation → nodes (SSH) | WireGuard → VLAN51 Bastion → internal nodes | Planned |

---

## Related Documentation

- [Infrastructure](infrastructure.md)
- [SOC Stack](soc-stack.md)
- [Internal PKI](pki.md)
- [SOP: Security Hardening](sop-sec-001.md)

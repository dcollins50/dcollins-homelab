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
    +— Cloudflare Tunnels (Web, Admin)
    +— Tailscale (break glass)
    |
TP-Link TL-SG108E (Managed Switch)
    |
    +— VLAN1  (Management)
    +— VLAN10 (SOC)
    +— VLAN20 (Services)
    +— VLAN30 (Trust Infrastructure)
    +— VLAN40 (Security Lab 1)
    +— VLAN41 (Security Lab 2)
    +— VLAN50 (DMZ1)
    +— VLAN51 (DMZ2 / SSH Bastion) [Planned]
    +— VLAN60 (Storage) [Planned]
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
- Aliases used extensively for host groups, VLAN subnets, and port groups, reducing rule duplication and centralizing updates to a single reference point rather than editing individual rules when an IP or port changes. This also limits what's exposed in any single rule and keeps the ruleset easier to audit.
- Anti-spoofing and RFC1918 blocking on WAN interface
- No inbound port forwarding rules on WAN (all external access via WireGuard, Cloudflare Tunnel, or Tailscale)

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
| pve-gateway | 10.0.0.10 |
| pve-services | 10.0.0.11 |
| pve-env1 | 10.0.0.12 |
| pve-env2 | 10.0.0.13 |

**Firewall policy:** Management VLAN access restricted to administrator subnet. No unsolicited inbound from other VLANs. Root SSH login is disabled on all Proxmox nodes; access requires a non-root account.

---

### VLAN10 — SOC

This VLAN hosts the SOC stack: detection and log analysis (ELK, Wazuh), alert automation (Shuffle SOAR), and incident case management (DFIR-IRIS).

| Host | IP |
|------|----|
| soc-stack (VM 600) | 10.0.10.10 |
| wazuh-manager (VM 601) | 10.0.10.11 |
| soar-host (VM 602) | 10.0.10.12 |
| pve-iris (VM 604) | 10.0.10.13 |

**Firewall policy:** SOC net has general outbound access on HTTP/HTTPS (80/443). DNS is restricted to OPNSense as the only permitted resolver; all other DNS destinations are blocked. LAN SSH is permitted to wazuh-manager only (TCP 22) for agent management, and wazuh-manager itself is permitted agentless SSH back to OPNSense. Wazuh agents on VLAN10 reach wazuh-manager on TCP/UDP 1514-1515. soar-host and pve-iris do not yet have dedicated firewall rules beyond the general SOC net outbound policy.

**Log flow:** OPNSense Suricata EVE JSON → Logstash (port 5144) → Elasticsearch. Wazuh agent alerts → Filebeat → Elasticsearch. Heimdall rsyslog → Logstash (port 5146) → Elasticsearch.

---

### VLAN20 — Services

| Host | IP |
|------|----|
| services-host (VM 200) | 10.0.20.30 |

Running on services-host: Vaultwarden, Internal NPM, Gitea, Stremio, Portainer, Uptime Kuma.

**Firewall policy:** Outbound internet permitted (HTTP/HTTPS via alias). DNS restricted to Heimdall's Pi-hole (192.168.100.1) as the only permitted resolver; all other DNS destinations blocked. NTP outbound permitted. Wazuh agents on VLAN20 reach wazuh-manager on TCP/UDP 1514-1515. Internal NPM has specific allowlisted reverse-proxy targets: Kibana (10.0.10.10:5601), Proxmox WebUI (all nodes, port 8006), services-host itself, pve-iris (443), and Authentik (9000, for SSO reverse proxy). Services VLAN can reach Authentik directly (port 9000) and soar-host (HTTP/HTTPS ports). Explicit block on Services-to-internal-VLANs traffic beyond these allowlisted paths (RFC1918 block), with general outbound internet still permitted.

---

### VLAN30 — Trust Infrastructure

| Host | IP |
|------|----|
| pve-ca-root (Root-CA) | 10.0.30.x |
| pve-ca-intermediate (Intermediate-CA) | 10.0.30.x |
| Authentik (LXC 2201) | 10.0.30.x |
| pve-int-stepca (LXC 511) | 10.0.30.x |

Root-CA and Intermediate-CA form the internal two-tier PKI, issuing certs for internal services. Authentik serves as the self-hosted IdP/SSO. pve-int-stepca runs step-ca in Docker, planned to take over Intermediate CA duties from the raw OpenSSL setup once configured.

---

### VLAN40 — Security Lab 1

| Host | IP |
|------|----|
| kali-attack (VM 300) | 10.99.0.x |
| metasploitable2 (VM 301) | 10.99.0.x |
| dvwa (VM 302) | 10.99.0.x |
| Jetson Orin Nano | 10.99.0.100 |

All hosts sit flat on VLAN40, so Kali can reach Metasploitable2 and DVWA directly with no additional firewall rule required.

**Firewall policy:** Restricted outbound internet for tool updates only. No route to production, services, trust infrastructure, SOC, or management VLANs.

---

### VLAN41 — Security Lab 2

| Host | IP |
|------|----|
| malware-win11 (VM 400) | 10.99.1.x |

**Firewall policy:** Fully air-gapped. No internet access, no route to or from any other VLAN.

---

### VLAN50 — DMZ1

| Host | IP |
|------|----|
| npm-dmz (public-facing reverse proxy) | 10.0.50.x |
| ntfy | 10.0.50.x |
| Self-Hosted Website | Planned |
| Stoat Messenger | Planned |

npm-dmz is live and handles reverse proxying from Cloudflare Tunnel to internal services (admin UI on port 81).

**Firewall policy:** Inbound 80/443 via Cloudflare Tunnel only. No route from DMZ to management, SOC, trust infrastructure, lab, or storage VLANs, unless explicitly allowlisted.

---

### VLAN51 — DMZ2

| Host | IP |
|------|----|
| SSH Bastion | Planned |

**Firewall policy:** Planned as the single SSH entry point for all internal nodes. Bastion IP will be the only permitted SSH source to internal VLANs once live; direct workstation-to-internal SSH will be blocked at that point.

---

### VLAN60 — Storage

Planned. No devices currently deployed. Will carry NAS traffic once built, with no internet access and management access restricted to the management VLAN.

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

Remote administration currently runs through Heimdall's WireGuard VPN, kept as one of three parallel access paths.

Cloudflare Tunnels handle both public web traffic and an admin path (Cloudflare Zero Trust / WARP client), routing to the planned SSH Bastion on VLAN51. The bastion's auth design requires two layers: WARP client enrollment via Authentik as IdP, plus step-ca issuing short-lived SSH certificates via its own OIDC login against Authentik, replacing a static SSH key.

Tailscale serves as a break-glass path if the bastion is down. Heimdall's WireGuard serves as a further backup if both the bastion and Tailscale are down. None of these three paths route to each other, Tailscale and WireGuard have no route to or from the bastion, and the bastion has no route to them.

---

## Traffic Flow Reference

| Flow | Path | Status |
|------|------|--------|
| Internet → internal services (admin) | Cloudflare Tunnel (Admin) / WireGuard / Tailscale | Live |
| Internet → public services | Cloudflare Tunnel (Web) → npm-dmz → DMZ | Live |
| Wazuh agents → manager | TCP/UDP 1514-1515, all VLANs → VLAN10 | Live |
| OPNSense logs → ELK | Syslog → Logstash port 5144 → Elasticsearch | Live |
| Suricata EVE → ELK | EVE JSON → Logstash → Elasticsearch | Live |
| Heimdall rsyslog → ELK | Syslog → Logstash port 5146 → Elasticsearch | Live |
| Internal services → PKI | Services (VLAN20) / trust infrastructure (VLAN30) → CA VMs | Live |
| Admin workstation → nodes | WireGuard / Tailscale / Cloudflare Zero Trust → VLAN1 management | Live |
| Admin workstation → nodes (SSH) | Bastion (VLAN51) → internal nodes, WARP + step-ca cert auth | Planned |

---

## Related Documentation

- [Infrastructure](infrastructure.md)
- [SOC Stack](soc-stack.md)
- [Internal PKI](pki.md)
- [Security Lab](security-lab.md)

---
name: Network
type: moc
status: live
---

Full network topology, VLAN design, and traffic flow. Default-deny posture, explicit allowlists for all inter-VLAN traffic. Per-VLAN detail lives in the Network/ folder notes (VLAN1 through VLAN60).

## Physical topology
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
    +- VLAN11 (AIops) [Planned]
    +— VLAN20 (Services)
    +— VLAN30 (Trust Infrastructure)
    +— VLAN40 (Security Lab 1)
    +— VLAN41 (Security Lab 2)
    +— VLAN50 (DMZ1)
    +— VLAN51 (DMZ2 / SSH Bastion)
    +— VLAN60 (Storage) [Planned]
```

## Diagram
![[network-diagram.png|700]]

> [!note] Diagram differences
> Finalized Sep 20, 2026. Two known differences: [[ntfy]] is drawn as its own box but is a container inside [[npm-dmz]], and [[pve-authtunnel]] (built Sep 21) is missing from VLAN50. Also not shown: [[VLAN11]] (planned), VLAN60 being planned rather than deployed, and the remote access order in [[remote-access-path-order]].

## VLANs
- [[VLAN1]] — Management
- [[VLAN10]] — SOC
- [[VLAN11]]: AIops (planned)
- [[VLAN20]] — Services
- [[VLAN30]] — Trust Infrastructure
- [[VLAN40]] — Security Lab 1
- [[VLAN41]] — Security Lab 2
- [[VLAN50]] — DMZ1
- [[VLAN51]] — DMZ2 / SSH Bastion
- [[VLAN60]] — Storage (planned)

## OPNSense firewall design principles
- Default-deny on all VLAN interfaces
- Explicit allowlists for all permitted inter-VLAN traffic
- Aliases used extensively for host groups, VLAN subnets, and port groups — centralizes updates to one reference point instead of editing individual rules, and limits what's exposed in any single rule
- Anti-spoofing and RFC1918 blocking on WAN interface
- No inbound port forwarding on WAN — all external access via WireGuard, Cloudflare Tunnel, or Tailscale

Suricata IDS/IPS runs on the WAN interface in detection-only mode. EVE JSON logs forward to ELK via Logstash. Promotion to inline blocking (IPS) is a pending hardening item.

## DNS architecture

> [!warning] Change pending
> This table describes the current state. [[dns-architecture]] (Sep 23, 2026) decided that Unbound on OPNsense becomes the resolver for the stack, Pi-hole stays for domain blocking only, and internal names move from `homelab.local` to `.internal`. Not started, tracked in [[dhcp-dns-rollout]]. Per-VLAN resolver settings already differ from this table: [[VLAN10]] uses OPNsense, [[VLAN20]] uses Pi-hole, and the rollout note says VLAN30 and VLAN51 use Unbound on their gateway (marked there as possibly stale).

| Layer | Role |
|-------|------|
| Pi-hole (Heimdall) | Network-wide DNS filtering/ad blocking, primary DNS for all VLANs |
| OPNSense Unbound | Query forwarding — `homelab.local` queries forwarded to internal DNS at 192.168.100.1 |
| Internal DNS records | `elasticsearch.homelab.local` → NPM IP |

## Remote access
Three parallel, non-intersecting paths. Priority is set in [[remote-access-path-order]]:
- Heimdall's WireGuard VPN (secondary, in progress)
- Cloudflare Tunnels — Web (public) and Admin (Zero Trust/WARP), the admin path is the primary remote admin path and routes to the SSH Bastion ([[pve-bastion]]) on VLAN51. Bastion auth requires two MFA layers: WARP client enrollment via Authentik as IdP, plus step-ca issuing short-lived SSH certs via its own OIDC login against Authentik, replacing a static SSH key.
- Tailscale — tertiary, break-glass path if the bastion and WireGuard are down

None of these three paths route to each other. The bastion has no route to Tailscale or WireGuard, and WireGuard has no route to the bastion. The tailnet reaching the bastion is acceptable, see [[tailscale-access-model]].

## Traffic flow reference
| Flow | Path | Status |
|------|------|--------|
| Internet → internal services (admin) | Cloudflare Tunnel (Admin) / WireGuard / Tailscale | Tailscale live, WireGuard in progress, bastion enrollment blocked |
| Internet → public services | Cloudflare Tunnel (Web) → npm-dmz → DMZ | Live |
| Wazuh agents → manager | TCP/UDP 1514-1515, all VLANs → VLAN10 | Live |
| OPNSense logs → ELK | Syslog → Logstash port 5144 → Elasticsearch | Live |
| Suricata EVE → ELK | EVE JSON → Logstash → Elasticsearch | Live |
| Heimdall rsyslog → ELK | Syslog → Logstash port 5146 → Elasticsearch | Live |
| Internal services → PKI | Services (VLAN20) / trust infrastructure (VLAN30) → CA VMs | Live |
| Admin workstation → nodes | WireGuard / Tailscale / Cloudflare Zero Trust → VLAN1 management | Tailscale live, WireGuard in progress, bastion enrollment blocked |
| Admin workstation → nodes (SSH) | Bastion (VLAN51) → internal nodes, WARP + step-ca cert auth | Bastion built, WARP enrollment blocked, cutover not started |

## Related
- [[Infrastructure]]
- [[SOC-Stack]]
- [[PKI]]
- [[Security-Lab]]

# SOP-SEC-001 — Homelab Security Hardening
## Pre-Internet Exposure Readiness

| Field | Value |
|-------|-------|
| Document ID | SOP-SEC-001 |
| Version | 1.0 |
| Status | Active |
| Effective Date | April 20, 2026 |
| Last Reviewed | April 20, 2026 |
| Review Cycle | Quarterly / After each major infrastructure change |
| Document Owner | Daniel Collins |
| Approved By | Daniel Collins |

---

## 1. Purpose

This Standard Operating Procedure defines the security hardening controls that must be implemented and verified on a homelab or small-scale self-hosted infrastructure before any service is exposed to the public internet. It provides a repeatable, auditable checklist to ensure a consistent security baseline across system builds, rebuilds, and major configuration changes.

---

## 2. Scope

This SOP applies to all nodes, virtual machines, containers, and network devices within the homelab environment, including hypervisors, firewalls, managed switches, always-on single-board computers, and any edge or gateway devices. It covers: SSH hardening, network segmentation (VLANs), container runtime security, DMZ and reverse proxy architecture, bastion host configuration, and pre-internet exposure readiness validation.

---

## 3. Roles and Responsibilities

| Role | Responsibilities |
|------|-----------------|
| System Administrator | Sole owner and operator. Responsible for executing, verifying, and documenting all controls within this SOP. Also responsible for reviewing and updating this document on the defined review cycle. |

---

## 4. Technology and Tool References

Where specific tools are referenced throughout this document (e.g., OPNSense, Proxmox, ELK Stack, Wazuh, Docker, Nginx Proxy Manager, Cloudflare), these represent the implementations used in this environment. Equivalent alternatives may be substituted at the administrator's discretion provided the stated security control objective is met. Tool selection should be documented in the environment's architecture record.

---

## 5. Definitions and Abbreviations

| Term | Definition |
|------|-----------|
| VLAN | Virtual Local Area Network — logical network segment enforced at the switch/firewall layer |
| DMZ | Demilitarized Zone — isolated network segment for public-facing services |
| SIEM | Security Information and Event Management — centralized log aggregation and alerting platform |
| IDS/IPS | Intrusion Detection / Prevention System — monitors and optionally blocks malicious traffic |
| PKI | Public Key Infrastructure — certificate authority hierarchy for issuing TLS certificates |
| NPM | Nginx Proxy Manager |
| SOP | Standard Operating Procedure |
| CA | Certificate Authority |
| WAN | Wide Area Network — the external/internet-facing interface |
| LAN | Local Area Network — the internal network |

---

## 6. Procedure

Complete each section in order. For each control item: check the box when the control is confirmed in place, enter your name or initials in the Verified By column, and record the date of verification. All items must reach a verified state before proceeding to Section 6 (Pre-Internet Exposure Final Checklist). Any item that cannot be completed must be documented with a compensating control or accepted risk justification before sign-off.

---

## Section 1 — SSH Hardening

**Objective:** Ensure all SSH-accessible nodes enforce key-based authentication, restrict access to authorized users, and implement automated brute-force protection.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Root login disabled on all nodes (`PermitRootLogin no`) | | |
| [ ] | Password authentication disabled (`PasswordAuthentication no`) | | |
| [ ] | Ed25519 (or RSA 4096) key pairs deployed to all managed nodes | | |
| [ ] | SSH key passphrases set on all private keys | | |
| [ ] | `AllowUsers` or `AllowGroups` directive configured on each host | | |
| [ ] | `MaxAuthTries` set to 3 or fewer | | |
| [ ] | `LoginGraceTime` set to 30 seconds or fewer | | |
| [ ] | Fail2ban (or equivalent) installed and active on all SSH-accessible nodes | | |
| [ ] | SSH warning banner configured (`Banner /etc/issue.net`) | | |
| [ ] | SSH access on all gateway/edge devices restricted to management network only | | |
| [ ] | SSH hardened on any always-on single-board or embedded devices | | |

---

## Section 2 — Firewall and VLAN Segmentation

**Objective:** Enforce network segmentation through VLAN isolation and default-deny firewall rules to limit lateral movement between trust zones.

### 2.1 — Firewall Rule Fundamentals

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Default-deny policy applied on all VLAN interfaces | | |
| [ ] | Management VLAN access locked to administrator IPs or subnets only | | |
| [ ] | Lab / testing VLANs have no route to production VLANs | | |
| [ ] | Highest-risk lab VLAN fully air-gapped (no internet, no lateral reach) | | |
| [ ] | Floating rules reviewed and documented | | |
| [ ] | Outbound NAT verified per VLAN — no unintended internet access | | |
| [ ] | Firewall aliases / address groups created for common host sets | | |
| [ ] | Anti-spoofing and RFC1918 blocking enforced on WAN interface | | |

### 2.2 — VLAN Segmentation Verification

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Hypervisor management UI accessible only from management subnet | | |
| [ ] | Internal services VLAN: outbound internet permitted, no unsolicited inbound from other VLANs | | |
| [ ] | Secondary services VLAN: outbound internet permitted, isolated from primary services VLAN | | |
| [ ] | DMZ VLAN: inbound 80/443 only; no route back to internal VLANs | | |
| [ ] | Storage VLAN: no internet access; management access only | | |
| [ ] | Edge/gateway device WAN path verified end-to-end | | |
| [ ] | Managed switch PVID and trunk port configuration verified | | |

### 2.3 — Firewall Service Hardening

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Firewall management UI not exposed on WAN interface | | |
| [ ] | Firewall management UI accessible via HTTPS only | | |
| [ ] | Firewall SSH access restricted to management VLAN only | | |
| [ ] | Firewall configuration exported and backed up after every change | | |
| [ ] | Automatic firmware/security updates enabled or scheduled | | |

---

## Section 3 — Container Runtime Hardening

**Objective:** Harden container daemon configuration and runtime behavior to minimize attack surface and prevent container escape or privilege escalation.

### 3.1 — Daemon Configuration

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Inter-container communication disabled by default (`icc: false`) | | |
| [ ] | User namespace remapping enabled (`userns-remap`) | | |
| [ ] | Live restore enabled to prevent daemon restart outages (`live-restore: true`) | | |
| [ ] | No-new-privileges set as daemon default | | |
| [ ] | Container daemon TCP socket not exposed; Unix socket only | | |

### 3.2 — Container Runtime Security

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | All production containers attached to named custom bridge networks (not default bridge) | | |
| [ ] | Containers running as non-root user where possible | | |
| [ ] | Capability drop applied (`cap_drop: ALL`) with only required capabilities re-added | | |
| [ ] | Stateless containers configured read-only (`read_only: true`) | | |
| [ ] | Memory and CPU resource limits defined on all production containers | | |
| [ ] | Container management UI secured with strong credentials and HTTPS | | |
| [ ] | Password manager / secrets service HTTPS-only via reverse proxy and internal CA | | |
| [ ] | Docker Compose files stored in version control | | |
| [ ] | Container images pinned to specific digest or version tags (not `:latest`) | | |
| [ ] | Container daemon socket not bind-mounted into unprivileged containers | | |

### 3.3 — Logging and Monitoring

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Container logs forwarded to centralized log management system | | |
| [ ] | Service uptime monitoring active on all production containers | | |

---

## Section 4 — Reverse Proxy and DMZ Architecture

**Objective:** Isolate all public-facing services in a dedicated DMZ network segment, routing external traffic through a reverse proxy and tunneled ingress only — no direct inbound port forwarding.

### 4.1 — DMZ VM Provisioning

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Architecture and traffic flow documented before implementation begins | | |
| [ ] | DMZ VM provisioned on isolated DMZ VLAN (recommended: 2 vCPU / 2 GB RAM minimum) | | |
| [ ] | DMZ VLAN firewall rules: outbound internet only by default | | |
| [ ] | DMZ VLAN to internal services VLAN rules: explicit allowlist only (no broad access) | | |
| [ ] | DMZ VLAN has no route to management, lab, or storage VLANs | | |

### 4.2 — Reverse Proxy Deployment

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Existing reverse proxy configuration exported prior to migration | | |
| [ ] | TLS certificate volumes preserved and migrated to new instance | | |
| [ ] | Fresh reverse proxy stack deployed on DMZ VM | | |
| [ ] | Proxy host configuration imported and verified in new instance | | |
| [ ] | All proxy hosts resolve correctly from DMZ VM | | |
| [ ] | Previous reverse proxy instance removed from internal VLAN after migration | | |

### 4.3 — Tunneled Ingress Setup

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Ingress tunnel created in provider dashboard | | |
| [ ] | Tunnel agent deployed on DMZ VM and configured to forward to reverse proxy | | |
| [ ] | DNS records set to tunnel-proxied (real IP not exposed) | | |
| [ ] | WAF managed rules enabled at ingress provider | | |
| [ ] | Confirmed: real public IP does not appear in any DNS record | | |
| [ ] | Confirmed: zero inbound port forwarding rules on firewall WAN interface | | |

---

## Section 5 — SSH Bastion Host

**Objective:** Establish a hardened single-entry-point for SSH access to all internal nodes, eliminating direct workstation-to-node SSH paths.

### 5.1 — Bastion VM Provisioning

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Bastion VM provisioned on VLAN51 | | |
| [ ] | Static IP assigned | | |
| [ ] | Hostname set and DNS record created | | |
| [ ] | OS hardened: SSH configured, Fail2ban installed, unnecessary services removed | | |

### 5.2 — Bastion SSH Configuration

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | `AllowTcpForwarding` enabled for ProxyJump support | | |
| [ ] | `X11Forwarding` disabled | | |
| [ ] | `ClientAliveInterval 300` / `ClientAliveCountMax 2` configured | | |
| [ ] | SSH audit logging configured and forwarding to SIEM | | |
| [ ] | Firewall rule: only bastion IP is permitted to SSH to internal nodes | | |

### 5.3 — Workstation SSH Client Configuration

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | SSH config file (`~/.ssh/config`) configured on administrator workstation with ProxyJump | | |
| [ ] | ProxyJump verified to each hypervisor node | | |
| [ ] | ProxyJump verified to all lab and services VMs | | |
| [ ] | Direct workstation to internal node SSH confirmed blocked at firewall | | |

---

## Section 6 — Pre-Internet Exposure Final Checklist

**Objective:** Confirm all security prerequisites are satisfied before any service is made accessible from the public internet.

### 6.1 — Network and Firewall Prerequisites

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Reverse proxy fully migrated to DMZ VLAN | | |
| [ ] | Zero inbound WAN port forwarding rules confirmed | | |
| [ ] | DMZ VLAN confirmed to have no route back to internal VLANs | | |
| [ ] | Real public IP confirmed absent from all DNS records | | |

### 6.2 — Intrusion Detection and Prevention

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | IDS/IPS installed on firewall | | |
| [ ] | Initially deployed in detection-only (IDS) mode | | |
| [ ] | Community or commercial ruleset enabled and active | | |
| [ ] | IDS logs forwarded to SIEM | | |
| [ ] | Rules reviewed and tuned before enabling IPS (inline blocking) mode | | |
| [ ] | IDS/IPS applied to WAN interface at minimum | | |

### 6.3 — Certificate and TLS

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Internal PKI operational (Root CA + Intermediate CA) for internal services | | |
| [ ] | Publicly trusted certificates obtained for public-facing services | | |
| [ ] | TLS 1.2 minimum enforced; TLS 1.0 and 1.1 disabled on all endpoints | | |
| [ ] | HSTS headers enabled on all public-facing endpoints | | |
| [ ] | Certificate expiry monitoring active | | |

### 6.4 — Access Control

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | All management UIs accessible via VPN only (not directly internet-exposed) | | |
| [ ] | Secrets manager admin panel disabled when not actively in use | | |
| [ ] | Any invite-only or restricted services configured to enforce that restriction before launch | | |

### 6.5 — Monitoring and Observability

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Centralized log management (SIEM) stack operational | | |
| [ ] | Firewall logs ingested into SIEM | | |
| [ ] | Hypervisor node logs ingested into SIEM (all nodes) | | |
| [ ] | Edge/gateway device logs ingested into SIEM | | |
| [ ] | Lab and services VM logs ingested into SIEM | | |
| [ ] | IDS/IPS alert logs ingested into SIEM | | |
| [ ] | Host-based intrusion detection agent deployed on critical VMs | | |
| [ ] | SIEM dashboards built and verified for key telemetry sources | | |
| [ ] | Bastion SSH audit logs ingested into SIEM | | |

### 6.6 — Backup and Recovery

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Hypervisor VM configurations backed up | | |
| [ ] | Persistent container volumes backed up | | |
| [ ] | Firewall configuration exported after every major change | | |
| [ ] | Internal CA private keys backed up to offline/encrypted storage | | |
| [ ] | Recovery runbooks documented for all critical systems | | |

---

## Sign-Off

By signing below, the System Administrator certifies that all controls in this SOP have been reviewed, implemented, and verified for the current environment build.

| Role | Name | Date |
|------|------|------|
| System Administrator | Daniel Collins | |

---

## Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | April 20, 2026 | Daniel Collins | Initial release |

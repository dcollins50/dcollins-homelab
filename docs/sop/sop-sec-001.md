# SOP-SEC-001 — Homelab Security Hardening
## Pre-Internet Exposure Readiness

| Field | Value |
|-------|-------|
| Document ID | SOP-SEC-001 |
| Version | 1.1 |
| Status | Active |
| Effective Date | April 20, 2026 |
| Last Reviewed | May 3, 2026 |
| Review Cycle | Quarterly / After each major infrastructure change |
| Document Owner | Daniel Collins |
| Approved By | Daniel Collins |

---

## 1. Purpose

This Standard Operating Procedure defines the security hardening controls that must be implemented and verified on a homelab or small-scale self-hosted infrastructure before any service is exposed to the public internet. It provides a repeatable, auditable checklist to ensure a consistent security baseline across system builds, rebuilds, and major configuration changes.

---

## 2. Scope

This SOP applies to all nodes, virtual machines, containers, and network devices within the homelab environment, including hypervisors, firewalls, managed switches, always-on single-board computers, and any edge or gateway devices. It covers: SSH hardening, network segmentation (VLANs), container runtime security, DMZ and reverse proxy architecture, bastion host configuration, VPN hardening, patch management, and pre-internet exposure readiness validation.

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
| MFA | Multi-Factor Authentication |
| KEX | Key Exchange — the algorithm negotiation phase of an SSH session |
| NTP | Network Time Protocol |

---

## 6. Procedure

Complete each section in order. For each control item: check the box when the control is confirmed in place, enter your name or initials in the Verified By column, and record the date of verification. All items must reach a verified state before proceeding to Section 6 (Pre-Internet Exposure Final Checklist). Any item that cannot be completed must be documented with a compensating control or accepted risk justification before sign-off.

---

## Section 1 — SSH Hardening

**Objective:** Ensure all SSH-accessible nodes enforce key-based authentication, restrict access to authorized users, implement automated brute-force protection, and limit the negotiated cipher surface.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Root login disabled on all nodes (`PermitRootLogin no`) | | |
| [ ] | Password authentication disabled (`PasswordAuthentication no`) | | |
| [ ] | Ed25519 (or RSA 4096) key pairs deployed to all managed nodes | | |
| [ ] | SSH key passphrases set on all private keys | | |
| [ ] | `AllowUsers` or `AllowGroups` directive configured on each host | | |
| [ ] | `MaxAuthTries` set to 3 or fewer | | |
| [ ] | `LoginGraceTime` set to 30 seconds or fewer | | |
| [ ] | `ClientAliveInterval 300` and `ClientAliveCountMax 2` configured on all nodes (not bastion only) | | |
| [ ] | Fail2ban (or equivalent) installed and active on all SSH-accessible nodes | | |
| [ ] | SSH warning banner configured (`Banner /etc/issue.net`) | | |
| [ ] | SSH access on all gateway/edge devices restricted to management network only | | |
| [ ] | SSH hardened on any always-on single-board or embedded devices | | |
| [ ] | Ciphers restricted to strong algorithms only (e.g., `chacha20-poly1305@openssh.com`, `aes256-gcm@openssh.com`) | | |
| [ ] | MACs restricted to strong algorithms only (e.g., `hmac-sha2-512-etm@openssh.com`, `hmac-sha2-256-etm@openssh.com`) | | |
| [ ] | KEX algorithms restricted (e.g., `curve25519-sha256`, `diffie-hellman-group16-sha512`) | | |
| [ ] | SSH key rotation schedule defined and documented (recommended: annually or after any suspected compromise) | | |

---

## Section 2 — Firewall and VLAN Segmentation

**Objective:** Enforce network segmentation through VLAN isolation and default-deny firewall rules to limit lateral movement between trust zones. Ensure both IPv4 and IPv6 traffic planes are controlled.

### 2.1 — Firewall Rule Fundamentals

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Default-deny policy applied on all VLAN interfaces | | |
| [ ] | Management VLAN access locked to administrator IPs or subnets only | | |
| [ ] | Lab / testing VLANs have no route to production VLANs | | |
| [ ] | Highest-risk lab VLAN fully air-gapped (no internet, no lateral reach) | | |
| [ ] | Floating rules reviewed and documented | | |
| [ ] | Outbound NAT verified per VLAN — no unintended internet access | | |
| [ ] | Outbound traffic per VLAN restricted to required ports and destinations only (egress filtering — not permit-all outbound) | | |
| [ ] | Firewall aliases / address groups created for common host sets | | |
| [ ] | Anti-spoofing and RFC1918 blocking enforced on WAN interface | | |

### 2.2 — IPv6 Controls

**Objective:** Ensure IPv6 does not create a parallel traffic path that bypasses IPv4 firewall rules.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | IPv6 explicitly disabled on all VLAN interfaces not requiring it, OR | | |
| [ ] | IPv6 firewall rules independently configured with default-deny policy mirroring IPv4 ruleset | | |
| [ ] | No SLAAC or DHCPv6 assignment occurring on any unintended interface | | |
| [ ] | IPv6 WAN exposure confirmed absent (or explicitly scoped and hardened) | | |

### 2.3 — DNS Policy

**Objective:** Prevent VLAN clients from bypassing the internal DNS resolver, which would evade internal monitoring and logging.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Internal DNS resolver designated for all VLANs | | |
| [ ] | Firewall rules block outbound DNS (UDP/TCP 53) to any destination except the internal resolver | | |
| [ ] | DNS-over-HTTPS bypass prevention considered (block known DoH providers at firewall or via DNS sinkhole) | | |
| [ ] | Internal resolver logs queries and forwards logs to SIEM | | |

### 2.4 — VLAN Segmentation Verification

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Hypervisor management UI accessible only from management subnet | | |
| [ ] | Internal services VLAN: outbound internet permitted to required ports only, no unsolicited inbound from other VLANs | | |
| [ ] | Secondary services VLAN: outbound internet permitted to required ports only, isolated from primary services VLAN | | |
| [ ] | DMZ VLAN: inbound 80/443 only; no route back to internal VLANs | | |
| [ ] | Storage VLAN: no internet access; management access only | | |
| [ ] | Edge/gateway device WAN path verified end-to-end | | |
| [ ] | Managed switch PVID and trunk port configuration verified | | |
| [ ] | Native VLAN on all trunk ports set to an unused VLAN ID (not VLAN1) to prevent VLAN hopping | | |
| [ ] | No access ports assigned to the native/unused VLAN | | |

### 2.5 — Firewall Service Hardening

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Firewall management UI not exposed on WAN interface | | |
| [ ] | Firewall management UI accessible via HTTPS only | | |
| [ ] | Firewall SSH access restricted to management VLAN only | | |
| [ ] | Firewall configuration exported and backed up after every change | | |
| [ ] | Automatic firmware/security updates enabled or scheduled | | |

---

## Section 3 — Container Runtime Hardening

**Objective:** Harden container daemon configuration and runtime behavior to minimize attack surface, prevent container escape or privilege escalation, and ensure secrets are not exposed through configuration files.

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
| [ ] | AppArmor or seccomp profile applied to all production containers | | |

### 3.3 — Image Security

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Container image vulnerability scan run prior to deployment (e.g., `trivy image` or `grype`) | | |
| [ ] | No critical or high CVEs present in deployed images without documented justification | | |
| [ ] | Image scan integrated into deployment workflow or documented as a manual gate | | |

### 3.4 — Secrets Management

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | No secrets (API keys, passwords, tokens) hardcoded in Compose files or Dockerfiles | | |
| [ ] | `.env` files excluded from version control via `.gitignore` | | |
| [ ] | `.env` file permissions restricted to owner read/write only (600) | | |
| [ ] | Secrets injected via Docker secrets, environment file references, or a secrets manager — not plaintext inline values | | |
| [ ] | Version control history audited to confirm no secrets were previously committed | | |

### 3.5 — Logging and Monitoring

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Container logs forwarded to centralized log management system | | |
| [ ] | Service uptime monitoring active on all production containers | | |

---

## Section 4 — Reverse Proxy and DMZ Architecture

**Objective:** Isolate all public-facing services in a dedicated DMZ network segment, routing external traffic through a reverse proxy and tunneled ingress only — no direct inbound port forwarding. Enforce rate limiting and security headers at the proxy layer.

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
| [ ] | Tunnel credential (token/secret) stored securely — not in plaintext config files committed to version control | | |
| [ ] | Tunnel credential rotation procedure documented | | |

### 4.4 — Proxy-Layer Security Controls

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Rate limiting configured at the reverse proxy for all public-facing hosts | | |
| [ ] | Security headers configured on all public-facing proxy hosts: | | |
| [ ] | &nbsp;&nbsp;`Strict-Transport-Security` (HSTS) enabled | | |
| [ ] | &nbsp;&nbsp;`X-Frame-Options: SAMEORIGIN` or `DENY` | | |
| [ ] | &nbsp;&nbsp;`X-Content-Type-Options: nosniff` | | |
| [ ] | &nbsp;&nbsp;`Referrer-Policy: strict-origin-when-cross-origin` (or stricter) | | |
| [ ] | &nbsp;&nbsp;`Content-Security-Policy` defined and tested per application | | |
| [ ] | Security headers verified externally (e.g., securityheaders.com) before go-live | | |

---

## Section 5 — SSH Bastion Host

**Objective:** Establish a hardened single-entry-point for SSH access to all internal nodes, eliminating direct workstation-to-node SSH paths. Require multi-factor authentication and log all sessions.

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
| [ ] | Session recording configured on bastion (e.g., `tlog`, `script`-based logging, or equivalent) | | |
| [ ] | Session recording output stored to a path the connecting user cannot modify or delete | | |
| [ ] | Firewall rule: only bastion IP is permitted to SSH to internal nodes | | |
| [ ] | MFA configured for bastion SSH access (e.g., TOTP via `libpam-google-authenticator` or hardware token) | | |

### 5.3 — External Access Path

**Objective:** Define and harden the path by which the administrator reaches the bastion from outside the network. This path must not expose the bastion's SSH port directly to the internet.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | External access to bastion gated behind VPN (WireGuard or equivalent) — SSH port not exposed on WAN | | |
| [ ] | VPN peer configuration for bastion access documented | | |
| [ ] | Confirmed: bastion SSH port not reachable from internet without active VPN connection | | |

### 5.4 — Workstation SSH Client Configuration

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | SSH config file (`~/.ssh/config`) configured on administrator workstation with ProxyJump | | |
| [ ] | ProxyJump verified to each hypervisor node | | |
| [ ] | ProxyJump verified to all lab and services VMs | | |
| [ ] | Direct workstation to internal node SSH confirmed blocked at firewall | | |

---

## Section 6 — VPN Hardening

**Objective:** Harden the VPN service used to gate administrative access, since all management UIs and the bastion host depend on it as a primary access control boundary.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | WireGuard (or equivalent) interface bound to a non-default port | | |
| [ ] | Unique key pair generated per peer — no shared keys | | |
| [ ] | `AllowedIPs` configured per peer to restrict reachable subnets to only what is required | | |
| [ ] | Peers with no ongoing need have been removed from the configuration | | |
| [ ] | VPN interface firewall rules configured to permit only required traffic from VPN subnet | | |
| [ ] | VPN config files stored securely — not in plaintext version control | | |
| [ ] | Peer key rotation schedule defined (recommended: annually or after any device loss/compromise) | | |
| [ ] | VPN connection logs forwarded to SIEM | | |

---

## Section 7 — Pre-Internet Exposure Final Checklist

**Objective:** Confirm all security prerequisites are satisfied before any service is made accessible from the public internet.

### 7.1 — Network and Firewall Prerequisites

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Reverse proxy fully migrated to DMZ VLAN | | |
| [ ] | Zero inbound WAN port forwarding rules confirmed | | |
| [ ] | DMZ VLAN confirmed to have no route back to internal VLANs | | |
| [ ] | Real public IP confirmed absent from all DNS records | | |

### 7.2 — Intrusion Detection and Prevention

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | IDS/IPS installed on firewall | | |
| [ ] | Initially deployed in detection-only (IDS) mode | | |
| [ ] | Community or commercial ruleset enabled and active | | |
| [ ] | IDS logs forwarded to SIEM | | |
| [ ] | Rules reviewed and tuned before enabling IPS (inline blocking) mode | | |
| [ ] | IDS/IPS applied to WAN interface at minimum | | |

### 7.3 — Certificate and TLS

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Internal PKI operational (Root CA + Intermediate CA) for internal services | | |
| [ ] | Publicly trusted certificates obtained for public-facing services | | |
| [ ] | TLS 1.2 minimum enforced; TLS 1.0 and 1.1 disabled on all endpoints | | |
| [ ] | HSTS headers enabled on all public-facing endpoints | | |
| [ ] | Certificate expiry monitoring active | | |

### 7.4 — Access Control

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | All management UIs accessible via VPN only (not directly internet-exposed) | | |
| [ ] | Secrets manager admin panel disabled when not actively in use | | |
| [ ] | Any invite-only or restricted services configured to enforce that restriction before launch | | |

### 7.5 — Time Synchronization

**Objective:** Accurate time across all nodes is foundational to log correlation, certificate validation, and SIEM alert accuracy.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | NTP service configured and active on all nodes and VMs | | |
| [ ] | All nodes synchronized to a consistent NTP source | | |
| [ ] | Time synchronization verified (`timedatectl status` or equivalent) on all nodes before go-live | | |
| [ ] | NTP service running on OPNSense and serving the internal network | | |

### 7.6 — Monitoring, Observability, and Alerting

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Centralized log management (SIEM) stack operational | | |
| [ ] | Firewall logs ingested into SIEM | | |
| [ ] | Hypervisor node logs ingested into SIEM (all nodes) | | |
| [ ] | Edge/gateway device logs ingested into SIEM | | |
| [ ] | Lab and services VM logs ingested into SIEM | | |
| [ ] | IDS/IPS alert logs ingested into SIEM | | |
| [ ] | VPN connection logs ingested into SIEM | | |
| [ ] | DNS query logs ingested into SIEM | | |
| [ ] | Host-based intrusion detection agent deployed on critical VMs | | |
| [ ] | SIEM dashboards built and verified for key telemetry sources | | |
| [ ] | Bastion SSH audit logs and session recordings ingested into SIEM | | |
| [ ] | Alert rules configured in SIEM for key events (e.g., failed logins, IDS triggers, cert expiry) | | |
| [ ] | Alert notification path verified end-to-end (e.g., email, webhook, SMS) — not just dashboard visibility | | |
| [ ] | Log retention policy defined and enforced (minimum retention period documented) | | |

### 7.7 — External Attack Surface Validation

**Objective:** Verify what is actually visible from the internet before go-live, using external tooling — not internal assumptions.

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | External port scan performed against public IP (e.g., `nmap` from external host or VPS) | | |
| [ ] | Shodan lookup performed against public IP — no unexpected services or historical exposures | | |
| [ ] | Only expected ports visible externally (typically none, or tunnel-provider IPs only) | | |
| [ ] | Real IP confirmed not exposed via any DNS record, email header, or certificate SAN leak | | |

### 7.8 — Backup and Recovery

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | Hypervisor VM configurations backed up | | |
| [ ] | Persistent container volumes backed up | | |
| [ ] | Firewall configuration exported after every major change | | |
| [ ] | Internal CA private keys backed up to offline/encrypted storage | | |
| [ ] | Recovery runbooks documented for all critical systems | | |
| [ ] | Backup restore test performed and result documented for each critical system | | |
| [ ] | Backup restore test scheduled on recurring cadence (recommended: quarterly) | | |

---

## Section 8 — Ongoing Maintenance and Patch Management

**Objective:** Ensure the security baseline established by this SOP does not degrade over time. Point-in-time hardening is not sufficient — this section defines the recurring controls required to maintain posture.

### 8.1 — Patch Cadence

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | OS and package updates reviewed and applied on all nodes on a defined schedule (recommended: weekly review, monthly application minimum) | | |
| [ ] | Container base images rebuilt or updated on a defined schedule | | |
| [ ] | Container image vulnerability scans re-run after any image update | | |
| [ ] | Firewall firmware updates reviewed and applied promptly (within 30 days of release for non-critical, within 72 hours for critical) | | |
| [ ] | WireGuard and other VPN software updated on patch cycle | | |

### 8.2 — Configuration Drift Detection

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | SSH hardening configuration verified on all nodes at each quarterly review | | |
| [ ] | Firewall ruleset reviewed for stale or overly permissive rules at each quarterly review | | |
| [ ] | VLAN segmentation verified (no new unintended routes) at each quarterly review | | |
| [ ] | Container runtime configuration reviewed for drift from documented baseline | | |

### 8.3 — Credential and Key Rotation

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | SSH key rotation performed on documented schedule | | |
| [ ] | VPN peer keys rotated on documented schedule | | |
| [ ] | Service account passwords and API tokens rotated on documented schedule | | |
| [ ] | Tunnel credentials (Cloudflare token or equivalent) rotated on documented schedule | | |

### 8.4 — SOP Review

| Status | Control Item | Verified By | Date |
|--------|-------------|-------------|------|
| [ ] | This SOP reviewed and updated after each major infrastructure change | | |
| [ ] | This SOP reviewed on defined quarterly cycle regardless of changes | | |
| [ ] | All revision history entries completed in the table below | | |

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
| 1.1 | May 3, 2026 | Daniel Collins | Added IPv6 controls, egress filtering, DNS policy, VLAN hopping prevention, SSH cipher/MAC/KEX hardening, SSH session timeout on all nodes, SSH key rotation policy, container image scanning, container secrets management, AppArmor/seccomp controls, proxy-layer rate limiting and security headers, tunnel credential security, bastion MFA, bastion session recording, bastion external access path, dedicated VPN hardening section, NTP synchronization, SIEM alerting and notification verification, log retention policy, external attack surface validation, backup restore testing, and ongoing patch management and drift detection section |

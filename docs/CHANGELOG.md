---
name: CHANGELOG
type: meta
status: live
---

# Changelog

All notable changes to the dcollins-homelab infrastructure are documented here. Organized newest-first, by date of implementation. Infrastructure changes, service deployments, security hardening, and documentation updates are all tracked.

---

## 2026-09-28

### Security
- Hardened a new the provider Debian 13 VPS ([[vps-redirector]], <public-ip-redacted>) to baseline: non-root sudo user `admin` with a dedicated `vps_key` ed25519 key, root SSH and password auth disabled (required removing a provider-shipped `/etc/ssh/sshd_config.d/99-ctrl.conf` drop-in that forced both back on), `nftables` default-deny allowing only SSH, plus `fail2ban` and `unattended-upgrades`. The box is a public redirector for the security lab; a Wazuh agent for it is still planned.

### Documentation
- New host note [[vps-redirector]]: the VPS as standing infrastructure, its redirector role, hardened state, the current Bore transport and why it falls short of the encryption/containment goals, and the planned WireGuard-on-OPNSense replacement. Offensive methodology deliberately left to the pentest-notes repo.
- New [[sop-harden-debian-vps]]: host-agnostic Debian VPS hardening SOP, dependency-ordered with a lockout-safety checkpoint at every SSH/firewall step, generalized from the the provider run.
- Linked the redirector into [[Security-Lab]].
- Added an idempotent bash bootstrap script (`Scripts/bash/harden-debian-vps.sh`, gitignored/local) that reproduces the whole hardening SOP in one re-runnable pass, with a lockout guard and the provider `99-ctrl.conf` handling; standardizes the SSH policy on a `00-hardening.conf` drop-in that sorts before provider drop-ins. Aligned [[sop-harden-debian-vps]] to this same drop-in method.
- Documented a script-based teardown/reprovision path for [[vps-redirector]] (reinstall OS then run the script), after confirming the provider's panel has no native snapshot. Chosen over image-restore to keep the box burnable; Vultr noted as the fallback if true snapshots are ever needed.
- Enriched [[vps-redirector]] with confirmed panel facts (Chicago node <redacted>, KVM, 512 MB / 30 GB / 500 GB/mo), a Rescue Mode lockout-recovery note, and a future item to explore the provider's API for scripting the reinstall trigger.
- Added [[vps-wireguard-tunnel]], an in-progress Projects checklist for building the WireGuard redirector tunnel (VPS to OPNSense, home-initiated, full-NAT relay to [[kali-attack]] scoped by an explicit deny). OPNSense GUI steps taken from current OPNSense docs, not memory. Parameters (subnet, ports, NAT design) flagged as proposed pending confirmation.
- Not written yet, by design: the WireGuard tunnel + OPNSense scoping doc, which is held until the tunnel is actually built (Bore still in place as of this session).

---

## 2026-09-26

### Security
- Diagnosed a reported "UDP flood on SSH login" from pve-env1 to soc-stack:5144. Root cause: an unscoped `*.* @10.0.10.10:5144` rsyslog forwarder on pve-env1, pve-services, and pve-env2 shipping every facility/priority, not an SSH-specific issue. Scoped to exclude `kern` and `daemon` facilities (evidence from tcpdump capture: kernel AppArmor audit noise and filebeat's own telemetry, not security signal) while keeping auth/authpriv/cron/etc. Full reasoning in [[rsyslog-forward-facility-scoping]]. pve-gateway excluded from the fix, its line 70 uses `@@` (TCP) not `@` (UDP), a separate unresolved inconsistency.

---

## 2026-09-25

### Infrastructure
- Node clocks standardized to UTC across 16 hosts: all 4 Proxmox hosts (pve-gateway, pve-services, pve-env1, pve-env2), Heimdall, 5 VMs (pve-iris, soc-stack, services-host, pve-ca-intermediate, soar-host), and 6 LXCs (authentik, pve-int-stepca, pve-bastion, pve-authtunnel, npm-dmz). wazuh-manager was already UTC. Prompted by discovering 3 different timezones in play during the rule 100004 confirmation work; full reasoning in [[standardize-node-clocks-utc]]. kali-attack, ubuntu-401, pve-ca-root, and OPNSense still pending.

### Security
- Wazuh suppression rule 100004 (rootcheck setuid false positive, added Sep 24/25) confirmed fleet-wide. Agent restart + rootcheck rescan run on pve-env1, pve-env2, and pve-gateway; zero rule 510 hits for the 6 setuid binaries on all three, matching pve-services. All four Proxmox nodes now confirmed clean.
- SSH key auth rolled out to 6 VMs previously on password auth: pve-iris, soc-stack, services-host, pve-ca-intermediate, soar-host, wazuh-manager. Verified with a BatchMode SSH test (fails rather than falling back to password), not just ssh-copy-id's own report. kali-attack, ubuntu-401, and pve-ca-root still pending (stopped at time of rollout). Decision and node clock findings that prompted this: [[standardize-node-clocks-utc]].

### Documentation
- New `Runbooks/linux/` folder: [[linux-diagnose-disk-usage]], the df/du/find drilldown pattern used independently to diagnose two separate wazuh-manager disk-full incidents. (A related proactive disk/log-shipping alerting idea was proposed multiple times in chat but explicitly deferred each time, not written up as a runbook since it was never actually built, still tracked as an open item.)
- New `Runbooks/vaultwarden/` and `Runbooks/gitea/` folders: [[vaultwarden-enable-mfa]] and [[gitea-enable-mfa]] (the latter noting TOTP only covers the web UI, not git-over-SSH on port 222).
- New `Runbooks/ssh/` folder: [[ssh-wsl-agent-setup]] and [[ssh-key-rollout-and-verify]], generalizing today's own SSH key rollout work (WSL agent quirks, host-key/sudo-TTY/ambiguous-prompt gotchas, and verifying key auth with a BatchMode test instead of trusting `ssh-copy-id`'s own report).
- [[uptime-kuma-wire-notification]] gained a section on the Kuma-status-vs-direct-ping decision for automated rechecks (chose `/metrics` parsing over direct ping because the monitor set includes Push-type monitors with nothing to ping).
- New `Runbooks/uptime-kuma/` folder: [[uptime-kuma-wire-notification]], covering default notification behavior and a real TLS gotcha wiring Kuma to Shuffle (self-signed cert, no skip-TLS option on the Webhook notification type; fixed by using Shuffle's plain HTTP endpoint instead of HTTPS).
- New `Runbooks/portainer/` folder: [[portainer-agent-socket-exposure]], the Docker-socket exposure fix found and applied on services-host during the Sep 13 incident investigation, never separately documented.
- [[tailscale-tailnet-policy]] gained a section on Pi-hole sinkholing `controlplane.tailscale.com`, the root cause of the whole tailnet sitting silently expired since June 2026 undetected.
- New `Runbooks/dfir-iris/` folder, first entry [[iris-test-alert-api]]: the working alert-creation API test (correct endpoint `/alerts/add`, required fields `alert_status_id`/`alert_customer_id`), reconstructed from a September 2026 chat session that was never written to the vault.
- 5th Wazuh runbook added: [[wazuh-ignore-rootcheck-path]], covering the agent-side `<rootcheck><ignore>` suppression pattern (distinct from the manager-side [[wazuh-add-custom-rule]] approach), reconstructed from an April 2026 Jetson `/dev/mqueue` false-positive fix that was never written to the vault.
- [[wazuh-restart-agent-confirm-scan]] corrected to include `agent_control -r -a` as a fleet-wide alternative to restarting agents one at a time — this would have saved a full manual pass across 4 nodes during the Sep 25 rule 100004 confirmation.
- [[wazuh-rootcheck-setuid-suppression]] (Decision) updated: verification status across all 4 Proxmox nodes was stale, now reflects the Sep 25 confirmation.
- 4th Wazuh runbook added: [[wazuh-enroll-agent]], reconstructed from an April 2026 chat session that was never written to the vault. Covers the real install/enroll steps for 8 agents, including a correction found mid-rollout (the working enrollment block needs no `ssl_manager_ca` line, contrary to the first attempt).
- 3 Wazuh runbooks created under `Runbooks/wazuh/` (new tool folder): [[wazuh-restart-agent-confirm-scan]], [[wazuh-add-custom-rule]] (extracted from the Phase 2 tuning doc, with a corrected validation command — `wazuh-analysisd -t` instead of the outdated `ossec-logtest -t`), and [[wazuh-check-agent-status]]. Source Project docs cross-referenced back to the new runbooks.
- CHANGELOG backfilled from session logs and incident records dating back to initial build in 2025.
- GitHub repo `docs/change-management/` folder created. Published GitHub-ready versions of `dhcp-dns-rollout` and `aiops-vlan-build`.

---

## 2026-09-24

### Security
- Wazuh suppression rule 100003 corrected: field name changed from `syscheck.path` to `file` so it actually matches FIM alerts under `/etc/pve/`. Eliminates approximately 15 false-positive alerts per day per Proxmox node.
- Wazuh suppression rule 100004 added: suppresses rootcheck trojan false positives on six setuid binaries (`chfn`, `chsh`, `passwd` in `/bin` and `/usr/bin`) using a `pcre2` match anchored to the trailing quote in `full_log`. Eliminates approximately 336 weekly false positives across all four Proxmox nodes.

### Documentation
- Vault audit completed. All repo docs migrated into Obsidian vault with updated frontmatter, naming conventions, and folder structure. Runbooks, SOPs, incidents, and sessions are now properly categorized.

---

## 2026-09-23

### Documentation
- Obsidian vault established as the single source of truth for all homelab documentation.
- Claude Desktop integrated with the vault via the filesystem MCP connector for live in-session documentation.
- Vault folder layout finalized: Hosts/, Network/, Runbooks/, SOPs/, Decisions/, Projects/, Incidents/, Sessions/, Archive/.

---

## 2026-09-21

### Infrastructure
- VLAN51 (DMZBastion) created on the TL-SG108E switch and in OPNSense (tag 51, gateway 10.0.51.1/24). Isolates the SSH bastion from the rest of VLAN50 DMZ.
- SSH bastion LXC build started on pve-services, targeted at 10.0.51.10. Native-install LXC, not Docker-in-LXC.
- Trunk ports 2 (pve-services) and 5 (OPNSense LAN) updated to carry VLAN51.

### Security
- step-ca bootstrapped with the real intermediate cert and key imported from VM 501. SHA256 fingerprint verified against pve-ca-root's actual cert.
- Authentik OIDC provisioner added to step-ca. Required: enabling step-ca remote admin management, adding a VLAN30 DNS override for `stepca.homelab.local`, creating a DEV-to-NPM firewall rule on port 443, and importing the homelab root CA cert into the step-ca container trust store.

---

## 2026-09-20

### Infrastructure
- PKI migrated from the flat management network onto VLAN30 (Trust Infrastructure). pve-ca-root is now at 10.0.30.21; pve-ca-intermediate is now at 10.0.30.22. Old 10.0.0.x addresses no longer reachable.
- Scoped workstation-to-host SSH firewall rules created for both CA hosts prior to migration using `pve_ca_root` and `pve_ca_intermediate` aliases.

---

## 2026-09-19

### Infrastructure
- pve-int-stepca (VMID 511) built on pve-services, VLAN30, 10.0.30.20. Debian 13, Docker-in-LXC with nesting enabled. Hosts the step-ca container that will replace the raw OpenSSL intermediate CA workflow.
- pve-iris (VMID 604) deployed on pve-env1, VLAN10 (SOC), 10.0.10.13. Full clone of VM 603 (docker-host.template), 4 vCPU / 8GB RAM. Runs DFIR-IRIS v2.4.29 via Docker Compose behind NPM with an internal PKI cert.
- Shuffle SOAR deployed on soar-host (VM 602, pve-env1, 10.0.10.12). All six containers running: frontend, backend, worker, orborus, security, opensearch.
- Debounce alerting workflow built in Shuffle: Uptime Kuma confirmed-DOWN triggers a 60-second wait, rechecks via Kuma's Prometheus `/metrics` endpoint, only creates an IRIS alert if the service is still down after the recheck.

### Security
- VLAN40/41 security lab split documented. malware-win11 (VM 400) confirmed on VLAN41 (air-gapped). Kali, Metasploitable2, DVWA, and the Jetson Orin Nano confirmed on VLAN40.

---

## 2026-09-15

### Security
- Wildcard DNS record removed from `homelab.example`. Replaced with explicit CNAMEs for the root domain and `ntfy` only. Eliminates accidental public discoverability of internal subdomains.
- Cloudflare hardening applied to `ntfy.homelab.example`: rate limiting (60 req/min/IP, 10-minute block), Cloudflare Managed WAF ruleset, Bot Fight Mode, US-only geo-blocking. Rate limiting and geo-blocking later extended to the root domain after observing scanner traffic.

### Fixed
- Authentik 504 Gateway Timeout: traced to a missing Services-to-DEV VLAN firewall rule that had silently been deleted. Rule rebuilt.
- Authentik 502 Bad Gateway: NPM proxy host scheme was incorrectly set to `https` instead of `http` for the Authentik backend connection.
- Pi-hole AAAA split-horizon DNS issue for `authentik.homelab.example` returning merged local and Cloudflare records simultaneously. Fixed with a `pihole-FTL` restart.

---

## 2026-09-14

### Infrastructure
- DEV VLAN (VLAN30, 10.0.30.0/24) repurposed as the Trust Infrastructure VLAN. Houses internal CAs, Authentik, and step-ca, separate from general app hosting on VLAN20.
- Authentik deployed as LXC 2201 on pve-services, VLAN30, 10.0.30.10. Sized for light-scale use (4-5 users). Implicit consent set as the default authorization flow for all internal applications.

### Fixed
- Jetson Filebeat (VLAN40, 10.99.0.100) had been silently blocked from reaching soc-stack log ingest on port 5044 since August 20. Fixed by adding a VLAN40 allow rule above the existing block rule. Created reusable aliases `SOC_Stack_Ingest` and `SOC_Stack_Log_Ports` (ports 5044, 5045, 5144, 5145).

---

## 2026-09-13

### Infrastructure
- Cluster brought back online after approximately a 2-month shutdown.
- npm-dmz deployed as a standalone LXC on pve-env1, VLAN50 DMZ, 10.0.50.69. Dedicated to public-facing Cloudflare Tunnel traffic, separate from the internal NPM instance on services-host.
- Cloudflare One account created on the free tier for Cloudflare Tunnel and Access setup.
- Tailscale re-authenticated across all 7 tailnet nodes after approximately 2.5 months of expired node keys.

### Security
- MFA (TOTP) enabled on Vaultwarden and Gitea.
- Portainer agent container rebuilt without a published host port. Now communicates with portainer-server exclusively via Docker internal DNS on `management-net`, eliminating the unnecessarily exposed Docker socket on the host network.
- Management-to-Services firewall rules scoped from wildcard to explicit port-level rules. New `services_host_admin_ports` alias (ports 22, 80, 81, 443, 3000, 222, 9443, 8080, 3001) derived from `ss -tulpn` output and Uptime Kuma's monitor list.

### Fixed
- Pi-hole/Unbound DNS forwarding loop: Pi-hole's `revServers` entry was forwarding all `homelab.local` queries to OPNSense Unbound, which forwarded them straight back to Pi-hole. Fixed by removing the `homelab.local` domain association from OPNSense Query Forwarding.
- Fleet-wide 15-20 second `sudo` and `hostname -f` delay: caused by the same DNS loop hanging on AAAA queries for `.homelab.local` hostnames. Same fix resolved both issues simultaneously. `sudo` dropped from 20-33 seconds to under 20ms.
- Wazuh Manager disk full (100%): `ossec-alerts-20.json` and `ossec-alerts-20.log` from August 20 had grown to 62GB unrotated. Truncated in place to avoid needing free disk. Added `<max_output_size>200M</max_output_size>` to `ossec.conf` and a cleanup cron to delete alert files older than 30 days.
- Tailscale re-auth hang on services-host: Pi-hole blocklist was sinkholing `controlplane.tailscale.com` as a false positive. Domain whitelisted.

---

## 2026-05-13

### Security
- Root SSH login disabled (`PermitRootLogin no`) confirmed across all four Proxmox nodes.

---

## 2026-05-02

### Security
- Logstash TLS certificate verification fully hardened. All five pipeline output blocks updated from `ssl_verification_mode => "none"` to `ssl_verification_mode => "full"` with the homelab intermediate CA cert deployed to `/etc/logstash/certs/`. Logstash now validates the Elasticsearch server cert against the internal PKI chain.
- CVE-2026-31431 patched across all Ubuntu VMs.

---

## 2026-04-19

### Security
- Wazuh agent rollout completed. Agents deployed across all target VMs and all four Proxmox nodes. OPNSense agentless SSH monitoring configured.

---

## 2026-04-18

### Fixed
- OPNSense agentless SSH monitoring for Wazuh was timing out on every check. Root cause was a missing `.passlist` entry on the Wazuh Manager. Fixed; agentless monitoring now operating cleanly.

---

## 2026-03-30

### Services
- Wazuh Manager v4.14.4 deployed as VM 601 on pve-env2, VLAN10 (SOC), 10.0.10.11. Manager-only deployment; the existing Elasticsearch stack handles indexing and dashboards. Filebeat ships Wazuh alerts to Elasticsearch. Initial `wazuh-alerts-4.x` index confirmed active.

### Security
- TLS enabled on Elasticsearch and Kibana using internal PKI certs. Elasticsearch cert issued with both DNS SAN (`elasticsearch.homelab.local`) and IP SAN (10.0.10.10) to support direct backend connections from Filebeat and Logstash. Kibana configured with the intermediate CA cert in `certificateAuthorities` for full chain validation.
- Logstash pipeline fixed following Elasticsearch TLS enablement. A drop filter added to `proxmox.conf` to discard oversized OPNSense log statistics messages that were blocking the pipeline worker.

### Fixed
- Heimdall stale route: `10.99.0.0/24 via 192.168.100.2` was present in the live routing table but missing from the NetworkManager persistent config. Corrected and persisted in the NM connection file.

---

## 2026-01 to 2026-02

### Infrastructure
- Eight-VLAN architecture implemented on OPNSense and TL-SG108E switch: VLAN1 (Management), VLAN10 (SOC), VLAN20 (Services), VLAN30 (Dev/Trust Infrastructure), VLAN40 (SecurityLab1), VLAN41 (SecurityLab2), VLAN50 (DMZ), VLAN60 (Storage).
- Security lab VMs moved to VLAN40 and VLAN41 with inter-VLAN traffic restricted by OPNSense firewall rules.

### Fixed
- Corosync stale IPs (February 23): stale 192.168.100.x addresses in `corosync.conf` caused cluster communication failures following the January subnet migration. All `ring0_addr` entries updated to 10.0.0.x addresses.

---

## 2026-01-09

### Infrastructure
- Full cluster subnet migration from 192.168.100.x to 10.0.0.x. All four Proxmox nodes and all VMs renumbered. Corosync cluster config updated across all nodes. `/etc/hosts` updated on all nodes.
- OPNSense deployed as the network gateway on pve-gateway. WAN on `192.168.100.250` (Heimdall-side), LAN on `10.0.0.1/24`. Replaced legacy pfSense VM (VM 100, destroyed).
- Internal two-tier PKI stood up: VM 500 (pve-ca-root, Root CA) and VM 501 (pve-ca-intermediate, Intermediate CA) on pve-services.
- ELK Stack 8.19.12 deployed on soc-stack (VM 600, pve-env2, VLAN10, 10.0.10.10). Kibana dashboards built for Suricata, Proxmox, Heimdall, and Jetson log sources.
- NPM (Nginx Proxy Manager) deployed on services-host (VM 200, pve-env1, VLAN20, 10.0.20.30). Serves all `*.homelab.local` domains with TLS using internal PKI certs.

### Services
- Gitea, Vaultwarden, Portainer, Uptime Kuma, and Stremio deployed on services-host (VM 200).

---

## 2025 Initial Build

### Infrastructure
- Four-node Proxmox cluster built: pve-gateway, pve-services, pve-env1, pve-env2 (HP Mini G3/G5/G6 units).
- OPNSense deployed as the network firewall and gateway on a dedicated G3 unit.
- Heimdall (Raspberry Pi 5) configured as WiFi bridge, Pi-hole DNS server, and WireGuard VPN server at 192.168.100.1.
- Jetson Orin Nano added on VLAN40 (10.99.0.100) for local AI inference.
- Security lab built on pve-gateway: Kali Linux (VM 300), Metasploitable2 (VM 301), DVWA (VM 302), malware-win11 air-gapped Windows 11 sandbox (VM 400).

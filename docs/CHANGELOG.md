---
name: CHANGELOG
type: meta
status: live
---

# Changelog

All notable changes to the dcollins-homelab infrastructure are documented here. Organized newest-first, by date of implementation. Infrastructure changes, service deployments, security hardening, and documentation updates are all tracked.

---

## 2026-11-09

### Security
- Origin exposure audit: no inbound path from the internet over IPv4 (no forwards on [[opnsense]] or the AT&T gateway) or IPv6 (AT&T gateway drops unsolicited inbound, tested from cellular). Public services are only reachable through the Cloudflare tunnels. Open items on [[heimdall]] (no host firewall, WireGuard path unreachable from outside, failed handshakes on eth0) were tabled. See [[session-2026-11-09-origin-exposure-audit]].

---

## 2026-11-08

### Infrastructure
- Proxmox package repositories fixed on all four nodes. [[pve-gateway]] and [[pve-services]] only had the Proxmox VE 8 (`bookworm`) No-Subscription repository left over from the 8 to 9 upgrade, so they were getting no Proxmox updates. [[pve-env1]] and [[pve-env2]] had the paid enterprise and Ceph enterprise repositories on with no subscription. Old and paid entries disabled, No-Subscription for `trixie` added on each.
- Cluster upgraded from Proxmox VE 9.1 to 9.2.26 (kernel 7.0.14-28-pve, QEMU 11, LXC 7.0, ZFS 2.4, corosync 3.2) on all four nodes, plus pending Debian security updates (openssl, curl, bind9, rsync and others). `wazuh-agent` and `filebeat` held on every node so the SOC pipeline is updated separately. All four rebooted; Daniel reports everything back online. See [[homelab-update-automation]].

### Decisions
- Homelab update automation will run Ansible through Semaphore, not Terraform, on a new LXC `ansible-lab` (1200, 10.0.30.x, [[pve-services]]). See [[semaphore-for-update-automation]].

### Documentation
- [[tailscale-tailnet-policy]] replaced with the live policy (Nov 7). It was missing the achilles share, pve-ansible and the Asscord staging VPS. Exceptions recorded in [[tailscale-access-model]].

---

## 2026-10-28

### Infrastructure
- Achilles SOC ([[achilles-soc-ansible]]): the `wazuh_agent` role now tells the agent which app security logs to read (`wazuh_agent_app_logs` per host, a managed `<localfile>` block in `ossec.conf`, agent restart on change). Run on Ichos and Hermes, each `ok=23 changed=2 failed=0`. Ichos app events confirmed arriving as alerts from the custom rules. Hermes: the agent is reading its files, first events not yet seen.
- Achilles SOC ([[achilles-soc-ansible]]): confirmed the app log folders from SPEC.md exist on both app servers and are being written, which replaces the 2026-10-26 note that they did not exist yet.
- Achilles SOC ([[achilles-soc-ansible]]): the rootcheck "trojaned file" alert on `/usr/bin/md5sum` (Ichos) was checked and judged a false positive, because the file belongs to the Rust coreutils package and `dpkg -V` is clean. Whether to turn off that check is undecided.

---

## 2026-10-26

### Infrastructure
- Achilles SOC ([[achilles-soc-ansible]]): custom Wazuh rules for the apps' security events (SPEC.md v1.0), in `roles/wazuh_stack/files/local_rules.xml`, mounted into the manager and deployable alone with the `wazuh_rules` tag. 17 test cases in `tests/wazuh-rules/test_rules.py` pass on the live manager. Deploy run `changed=4`, rerun `changed=0`, both agents Active after the manager restart.
- Achilles SOC ([[achilles-soc-ansible]]): confirmed SSH sign-ins and sudo use reach the SOC from Ichos and Hermes through the journal, with no changes. Auditd is not installed on Ichos; whodata deferred.
- Achilles SOC ([[achilles-soc-ansible]]): SPEC.md is still a draft awaiting approval, and none of its log folders exist on the app servers yet. Per-app log pipelines wait on the collaborator.

---

## 2026-10-25

### Infrastructure
- Achilles SOC ([[achilles-soc-ansible]]): `playbooks/site.yml` runs Docker and the Wazuh stack, with accounts and health checks, in one command and one vault prompt (`--tags all,wazuh_up`). Dry run and real run on the built server both `changed=0`. Run on a blank server in the second wipe the same day, see below. Shortcuts `soc-sync`, `soc-ready` and `soc-ship` added, see [[ansible-shortcuts]].
- Achilles SOC ([[achilles-soc-ansible]]): OS account on/off switch in the `baseline` role. `enabled: false` locks an account and keeps its home, adding `remove: true` deletes it, and the Ansible service account cannot be switched. Proved on the live server with collab-user: lock, rerun `changed=0`, unlock, rerun `changed=0`. New runbook [[ansible-switch-os-account]].
- Achilles SOC ([[achilles-soc-ansible]]): named dashboard accounts for [admin-user] and collab-user (admin, one vault password each, `enabled` switch), made through the indexer's user API. Proved on the live server: create, rerun `changed=0`, off deletes the account, on recreates it. Script 05 of the collaborator's kit turned out not to contain this; it is new work.
- Achilles SOC wipe and rerun test PASSED ([[sop-rebuild-achilles-soc]]): Hostinger reinstall, then baseline, Tailscale, lockdown, Docker and the Wazuh stack from the repo. Certificates were signed by the vault root CA, the rerun changed nothing, the public ports timed out, and one reboot brought all three containers back with the same bindings. The generator task is now skipped in dry runs.
- Achilles SOC second wipe test PASSED ([[sop-rebuild-achilles-soc]]): a Hostinger reinstall, then baseline, Tailscale and lockdown, then `playbooks/site.yml` for Docker and Wazuh on the blank server. Real run `ok=70 changed=28 failed=0`, rerun `changed=0`, public ports timed out, the admin login and the Wazuh app worked, and one reboot returned all three containers with the same bindings. The `site.yml` dry run fails on a blank server at the account password check, so use only the real run there.
- Achilles SOC build ([[achilles-soc-ansible]]): handlers now flush before the Wazuh stack starts; the Wazuh root CA is kept in a second vault file and restored on rebuild; the certificate tool is pinned by hash and no longer downloaded at run time; the Ansible collections are pinned; the repo README is current. The CA on the server was checked against the vault. The wipe and rerun test is recorded in the entry above.
- Achilles SOC ([[achilles-soc-ansible]]): the app servers Ichos and Hermes joined the Achilles tailnet with `playbooks/02-tailscale-apps.yml` and a separate `inventories/app-servers`, tag `tag:app-server`. Each got one dedicated `ansible` account (key and passwordless sudo, by hand at the console). Tailnet SSH from the control node works on both, and both are blocked from the SOC server on 443 by the policy. Hermes needed `AllowUsers ansible` added through a sshd drop-in. Public SSH left open.
- Achilles SOC ([[achilles-soc-ansible]]): tailnet policy now grants `tag:app-server` to `tag:soc-server` on tcp:1514 and tcp:1515, with tests. Tested from Ichos: 1514 and 1515 open, 22 and 443 still blocked. The policy lives in the admin console only.
- Achilles SOC ([[achilles-soc-ansible]]): checked that agent enrollment needs a password (an attempt with none was refused, no agent created) and that the default passwords for the built in `wazuh-wui` and `wazuh` API users are rejected.
- Achilles SOC ([[achilles-soc-ansible]]): researched Wazuh 4.14 agent support on Ubuntu 26.04 (all three servers run it). Basics pass in Wazuh's own testing, FIM whodata (eBPF) failed and SCA has known issues. Decision: stay on Wazuh 4.14.9, test the agent on one server first.
- Achilles SOC ([[achilles-soc-ansible]]): researched FIM whodata on Ubuntu 26.04. The known failure is in the eBPF provider. The audit provider is the default and is the plan, pending a test on one server.
- Achilles SOC ([[achilles-soc-ansible]]): new `wazuh_agent` role. Ichos and Hermes enrolled as `ichos-vps` and `hermes-vps`, both Active on the manager, reruns `changed=0`. The enrollment password is in a new app-servers vault file.

### Documentation
- Added the missing `allowed_signers` steps to [[ansible-setup-control-node]], the second vault file and the vault transfer steps to [[ansible-vault-edit-secrets]], the CA restore to [[ansible-run-wazuh-stack]], and the end of session agent step to [[ansible-load-ssh-agent-key]].
- [[sop-rebuild-achilles-soc]]: added a before-you-wipe checklist, host key clearing on every client, the tailnet address change, and what to watch in the Wazuh run. Now marked passed.
- [[sop-rebuild-achilles-soc]]: second wipe results, a note on blank console output and checking the host key over the tailnet, and the dry run limit of `site.yml`.

---

## 2026-10-24

### Infrastructure
- Added [[pve-ansible]] (LXC 1100 on [[pve-env1]]), the Ansible control node for the Achilles SOC build. Project: [[achilles-soc-ansible]].

### Documentation
- Added the Ansible runbooks in Runbooks/ansible/: [[ansible-setup-control-node]], [[ansible-run-playbook]], [[ansible-run-wazuh-stack]], [[ansible-vault-edit-secrets]], [[ansible-load-ssh-agent-key]], [[ansible-verify-signed-commit]].
- Added [[sop-rebuild-achilles-soc]] (draft, full rebuild not yet tested) and the decision [[docker-published-ports-bypass-ufw]]. All of these are marked `publish: false`.

---

## 2026-10-23

### Infrastructure
- Tailnet policy: added host alias `[admin-user]-worktop` (100.x.x.x), a grant to [[nimbus]] on `tcp:443` only, and a test for it. The laptop was enrolled but blocked by default deny. Login to nimbus from it confirmed. Updated [[tailscale-access-model]], [[Tailscale]] and the policy copy in [[tailscale-tailnet-policy]].

---

## 2026-10-10

### Infrastructure
- Workstation network recovered after a Windows update (KB5121794, Oct 9): IP conflict on 192.168.1.x resolved by moving Wi-Fi to DHCP, Ethernet 2 interface metric set manually to 200 and Wi-Fi to 30, homelab persistent routes re-added (10.0.0.0/16, 10.99.0.0/24, 192.168.100.0/24 via 10.0.0.x). See [[incident-2026-10-10-workstation-network-drift]].
- AT&T gateway: Fixed allocation (DHCP reservation) created for the workstation's Wi-Fi MAC to 192.168.1.x. The unknown randomized-MAC device that had taken `.241` was moved to a Fixed 192.168.1.x and is still unidentified. Workstation confirmed on 192.168.1.x, Dhcp, Preferred.
- Pending: reboot test of metrics and routes, identify the device at 192.168.1.x.

### Documentation
- Added [[windows-restore-dual-homed-network]] (Runbooks/windows/) with a baseline capture and a step-by-step recovery to the target state.

---

## 2026-10-03

### Infrastructure
- [[pve-bastion]] WARP connection fixed: IPv6 enabled on the WARP interface only (`default` set to 0 in `/etc/sysctl.d/99-disable-ipv6.conf` and `/etc/sysctl.conf`, `all` and `eth0` still 1). Stayed connected through a container reboot. See [[session-2026-10-03-bastion-warp-ipv6]] and [[cloudflare-warp-mesh-node-headless]].
- [[pve-bastion]] host firewall added: default drop on input, SSH only on the WARP interface, port 22 on `eth0` dropped, outbound open. Survives reboot, WARP unaffected. See [[bastion-host-firewall]].
- Workstation enrolled in the Cloudflare One Client (Windows 11, alongside Tailscale). Ping and SSH to [[pve-bastion]] over the Mesh work. See [[cloudflare-warp-windows-workstation-mesh-access]].
- Cloudflare Gateway network policies: added "Allow admin SSH to bastion" (number 1) and enabled "Default deny for private traffic" (number 2). Tested with the Allow policy off and on. ICMP is not gated. See [[bastion-host-firewall]] and [[session-2026-10-03-bastion-warp-ipv6]].

### Infrastructure (later, SSH certificates)
- step-ca ([[pve-int-stepca]]) now has an SSH CA: SSH host and user CA keys created and an `ssh` block added to `ca.json`. Authentik provisioner secret re-entered and the loopback redirect URI added to the Authentik `step-ca` provider.
- [[pve-bastion]] sshd trusts the step-ca SSH user CA, with `[admin-user]` mapped to the principal `user@example.com`, and password login is off. Tested over the Mesh. See [[bastion-ssh-certificate-login]] and [[session-2026-10-03-stepca-ssh-ca]].

- Temporary OPNsense `workstation` to `bastion` port 22 rule reported removed by Daniel (interface tab and a follow-up test not recorded).

### Documentation
- Added the planning note [[admin-portal]] for a tile-style admin page behind the bastion.
- Added an exception for WARP hosts to the IPv6 step in [[proxmox-lxc-build-pattern]].

---

## 2026-10-02

### Infrastructure
- Fixed the Authentik login failure in WARP enrollment: the `authentik` proxy host in Nginx Proxy Manager now has larger proxy buffers ([[npm-fix-502-upstream-header-too-big]]).
- Enabled `nesting=1` on [[pve-authtunnel]] (CT 2100) to restore journald ([[proxmox-lxc-journald-credentials-nesting]]).
- [[pve-bastion]] enrolled in Cloudflare Zero Trust as Mesh node `bastion-node` (Mesh IP only). Not yet stable, the WARP DNS proxy fails with IPv6 disabled. See [[session-2026-10-02-bastion-warp-enrollment]] and [[cloudflare-warp-mesh-node-headless]].

### Documentation
- Sanitizer now runs automatically. `Scripts/python/publish-sync.py` watches the vault, re-sanitizes notes after they stop changing, writes to `docs/`, and stages the result with `git add`. Notes with `publish: false` stay out, notes with review items are held. Optional Obsidian trigger via the Shell commands plugin. See [[doc-sanitization-policy]] and [[sanitize-automate-publish]].

---

## 2026-10-01

### Documentation
- Added an on-demand sanitizer for publishing notes: `Scripts/python/sanitize-doc.py` with local rules in `Scripts/sanitize-rules.json`. It masks IPs, Tailscale names, public addresses, emails, usernames and secrets, writes only to the repo's `docs/` folder, and refuses to write inside the vault. Policy in [[doc-sanitization-policy]], how to run it in [[sanitize-publish-doc]].

---

## 2026-09-30

### Infrastructure
- [[nimbus]] (CT 3400, 10.0.50.x) is live: Nextcloud 34 on Apache, PHP 8.4 and PostgreSQL 17 in the DMZ ([[VLAN50]]) on [[pve-env1]], shared to one outside user over Tailscale. HTTPS through `tailscale serve`, container firewall accepts only Tailscale traffic on 443, two-factor enforced, accounts `collab-user` and `[admin-user]` created. The admin account `[nimbus-admin]` is disabled (re-enable with `occ user:enable` when needed).
- Added a weekly read-only snapshot of nimbus's data disk on [[pve-env1]] (`/usr/local/bin/nimbus-data-snapshot`, cron `30 3 * * 0`) and widened `/usr/local/bin/devgit-backup-send` so it also serves nimbus container backups.

### Security
- New pull key `nimbus-data-pull` on `admin@pve-env1`, forced to `sudo /usr/bin/rrsync -ro /mnt/nimbus-data-snap/data`, with a sudoers rule allowing only that command. Read-only, one folder, no shell.
- Tailnet policy: `nimbus` grants on `tcp:80` and `tcp:443` with tests that it cannot reach other machines. Updated [[tailscale-access-model]].

### Backups
- Workstation scheduled tasks `Pull nimbus data` (rsync in WSL to `D:\Backups\cloud-files`, 28-day history of changed and deleted files) and `Pull nimbus backup` (Proxmox container backup to `D:\Backups\nimbus`, keep 3). Both tested. Restore not tested yet. See [[nimbus-backup-design]].

### Documentation
- New [[nimbus-backup-design]], [[session-2026-09-30-cloud-files-build]]. Rewrote [[nimbus]] for the live state, updated [[cloud-files-design]], [[sop-cloud-files-build]], [[dev-git-backup-strategy]], [[pve-env1]] and [[VLAN50]]. Added `Scripts\bash\pull-nimbus-data.sh`, `Scripts\bash\nimbus-data-snapshot.sh` and `Scripts\PowerShell\pull-nimbus-backup.ps1`.

---

## 2026-09-29

### Documentation
- Corrected the [[racknerd-vps]] API note. Verified against SolusVM's own docs that RackNerd's API (SolusVM client API, key+hash, scoped per-VPS) supports only Boot, Reboot, Shutdown, and Status. It cannot reinstall, so the earlier "script the reinstall trigger" idea is dead; it also exposes no bandwidth, console, or hostname. Power-only, so a leaked key is a relay-drop (DoS) risk, not a data/rebuild risk.

---

## 2026-09-28

### Security
- Hardened a new RackNerd Debian 13 VPS ([[racknerd-vps]], [vps-public-ip]) to baseline: non-root sudo user `[admin-user]` with a dedicated `racknerd_vps` ed25519 key, root SSH and password auth disabled (required removing a provider-shipped `/etc/ssh/sshd_config.d/99-ctrl.conf` drop-in that forced both back on), `nftables` default-deny allowing only SSH, plus `fail2ban` and `unattended-upgrades`. The box is a public redirector for the security lab; a Wazuh agent for it is still planned.

### Documentation
- New host note [[racknerd-vps]]: the VPS as standing infrastructure, its redirector role, hardened state, the current Bore transport and why it falls short of the encryption/containment goals, and the planned WireGuard-on-OPNSense replacement. Offensive methodology deliberately left to the pentest-notes repo.
- New [[sop-harden-debian-vps]]: host-agnostic Debian VPS hardening SOP, dependency-ordered with a lockout-safety checkpoint at every SSH/firewall step, generalized from the RackNerd run.
- Linked the redirector into [[Security-Lab]].
- Added an idempotent bash bootstrap script (`Scripts/bash/harden-debian-vps.sh`, gitignored/local) that reproduces the whole hardening SOP in one re-runnable pass, with a lockout guard and RackNerd `99-ctrl.conf` handling; standardizes the SSH policy on a `00-hardening.conf` drop-in that sorts before provider drop-ins. Aligned [[sop-harden-debian-vps]] to this same drop-in method.
- Documented a script-based teardown/reprovision path for [[racknerd-vps]] (reinstall OS then run the script), after confirming RackNerd's panel has no native snapshot. Chosen over image-restore to keep the box burnable; Vultr noted as the fallback if true snapshots are ever needed.
- Enriched [[racknerd-vps]] with confirmed panel facts (Chicago node CHI136KVM, KVM, 512 MB / 30 GB / 500 GB/mo), a Rescue Mode lockout-recovery note, and a future item to explore RackNerd's API for scripting the reinstall trigger.
- Added [[racknerd-wireguard-tunnel]], an in-progress Projects checklist for building the WireGuard redirector tunnel (VPS to OPNSense, home-initiated, full-NAT relay to [[kali-attack]] scoped by an explicit deny). OPNSense GUI steps taken from current OPNSense docs, not memory. Parameters (subnet, ports, NAT design) flagged as proposed pending confirmation.
- Not written yet, by design: the WireGuard tunnel + OPNSense scoping doc, which is held until the tunnel is actually built (Bore still in place as of this session).

---

## 2026-09-26

### Security
- Diagnosed a reported "UDP flood on SSH login" from pve-env1 to soc-stack:5144. Root cause: an unscoped `*.* @10.0.10.x:5144` rsyslog forwarder on pve-env1, pve-services, and pve-env2 shipping every facility/priority, not an SSH-specific issue. Scoped to exclude `kern` and `daemon` facilities (evidence from tcpdump capture: kernel AppArmor audit noise and filebeat's own telemetry, not security signal) while keeping auth/authpriv/cron/etc. Full reasoning in [[rsyslog-forward-facility-scoping]]. pve-gateway excluded from the fix, its line 70 uses `@@` (TCP) not `@` (UDP), a separate unresolved inconsistency.

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
- VLAN51 (DMZBastion) created on the TL-SG108E switch and in OPNSense (tag 51, gateway 10.0.51.x/24). Isolates the SSH bastion from the rest of VLAN50 DMZ.
- SSH bastion LXC build started on pve-services, targeted at 10.0.51.x. Native-install LXC, not Docker-in-LXC.
- Trunk ports 2 (pve-services) and 5 (OPNSense LAN) updated to carry VLAN51.

### Security
- step-ca bootstrapped with the real intermediate cert and key imported from VM 501. SHA256 fingerprint verified against pve-ca-root's actual cert.
- Authentik OIDC provisioner added to step-ca. Required: enabling step-ca remote admin management, adding a VLAN30 DNS override for `stepca.homelab.local`, creating a DEV-to-NPM firewall rule on port 443, and importing the homelab root CA cert into the step-ca container trust store.

---

## 2026-09-20

### Infrastructure
- PKI migrated from the flat management network onto VLAN30 (Trust Infrastructure). pve-ca-root is now at 10.0.30.x; pve-ca-intermediate is now at 10.0.30.x. Old 10.0.0.x addresses no longer reachable.
- Scoped workstation-to-host SSH firewall rules created for both CA hosts prior to migration using `pve_ca_root` and `pve_ca_intermediate` aliases.

---

## 2026-09-19

### Infrastructure
- pve-int-stepca (VMID 511) built on pve-services, VLAN30, 10.0.30.x. Debian 13, Docker-in-LXC with nesting enabled. Hosts the step-ca container that will replace the raw OpenSSL intermediate CA workflow.
- pve-iris (VMID 604) deployed on pve-env1, VLAN10 (SOC), 10.0.10.x. Full clone of VM 603 (docker-host.template), 4 vCPU / 8GB RAM. Runs DFIR-IRIS v2.4.29 via Docker Compose behind NPM with an internal PKI cert.
- Shuffle SOAR deployed on soar-host (VM 602, pve-env1, 10.0.10.x). All six containers running: frontend, backend, worker, orborus, security, opensearch.
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
- Authentik deployed as LXC 2201 on pve-services, VLAN30, 10.0.30.x. Sized for light-scale use (4-5 users). Implicit consent set as the default authorization flow for all internal applications.

### Fixed
- Jetson Filebeat (VLAN40, 10.99.0.x) had been silently blocked from reaching soc-stack log ingest on port 5044 since August 20. Fixed by adding a VLAN40 allow rule above the existing block rule. Created reusable aliases `SOC_Stack_Ingest` and `SOC_Stack_Log_Ports` (ports 5044, 5045, 5144, 5145).

---

## 2026-09-13

### Infrastructure
- Cluster brought back online after approximately a 2-month shutdown.
- npm-dmz deployed as a standalone LXC on pve-env1, VLAN50 DMZ, 10.0.50.x. Dedicated to public-facing Cloudflare Tunnel traffic, separate from the internal NPM instance on services-host.
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
- Wazuh Manager v4.14.4 deployed as VM 601 on pve-env2, VLAN10 (SOC), 10.0.10.x. Manager-only deployment; the existing Elasticsearch stack handles indexing and dashboards. Filebeat ships Wazuh alerts to Elasticsearch. Initial `wazuh-alerts-4.x` index confirmed active.

### Security
- TLS enabled on Elasticsearch and Kibana using internal PKI certs. Elasticsearch cert issued with both DNS SAN (`elasticsearch.homelab.local`) and IP SAN (10.0.10.x) to support direct backend connections from Filebeat and Logstash. Kibana configured with the intermediate CA cert in `certificateAuthorities` for full chain validation.
- Logstash pipeline fixed following Elasticsearch TLS enablement. A drop filter added to `proxmox.conf` to discard oversized OPNSense log statistics messages that were blocking the pipeline worker.

### Fixed
- Heimdall stale route: `10.99.0.0/24 via 192.168.100.x` was present in the live routing table but missing from the NetworkManager persistent config. Corrected and persisted in the NM connection file.

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
- OPNSense deployed as the network gateway on pve-gateway. WAN on `192.168.100.x` (Heimdall-side), LAN on `10.0.0.x/24`. Replaced legacy pfSense VM (VM 100, destroyed).
- Internal two-tier PKI stood up: VM 500 (pve-ca-root, Root CA) and VM 501 (pve-ca-intermediate, Intermediate CA) on pve-services.
- ELK Stack 8.19.12 deployed on soc-stack (VM 600, pve-env2, VLAN10, 10.0.10.x). Kibana dashboards built for Suricata, Proxmox, Heimdall, and Jetson log sources.
- NPM (Nginx Proxy Manager) deployed on services-host (VM 200, pve-env1, VLAN20, 10.0.20.x). Serves all `*.homelab.local` domains with TLS using internal PKI certs.

### Services
- Gitea, Vaultwarden, Portainer, Uptime Kuma, and Stremio deployed on services-host (VM 200).

---

## 2025 Initial Build

### Infrastructure
- Four-node Proxmox cluster built: pve-gateway, pve-services, pve-env1, pve-env2 (HP Mini G3/G5/G6 units).
- OPNSense deployed as the network firewall and gateway on a dedicated G3 unit.
- Heimdall (Raspberry Pi 5) configured as WiFi bridge, Pi-hole DNS server, and WireGuard VPN server at 192.168.100.x.
- Jetson Orin Nano added on VLAN40 (10.99.0.x) for local AI inference.
- Security lab built on pve-gateway: Kali Linux (VM 300), Metasploitable2 (VM 301), DVWA (VM 302), malware-win11 air-gapped Windows 11 sandbox (VM 400).

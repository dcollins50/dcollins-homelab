---
name: session-2026-09-24-vault-audit
type: session
date: 2026-09-24
---

# Session Log: September 24, 2026: Vault Audit and Mechanical Fixes

**Scope:** Audited the vault against [[Conventions]] and the Obsidian Flavored Markdown rules, then fixed the mechanical problems. The Obsidian CLI could not be used because it needs a running Obsidian app and the session ran in a sandbox. All checks were done by reading files through the filesystem connector, so there are no backlink, orphan or unresolved-link counts.

## Coverage
- Read in full: Conventions, all Hosts/ notes, all Network/ notes, Welcome, Tailscale, Infrastructure, Network, PKI, Security-Lab, SOC-Stack, `sop-cloudflare-tunnel-implementation`, `aiops-vlan`, `dns-architecture`, `dhcp-dns-rollout`.
- Header only: the other SOPs, the Projects notes, 3 of 14 Incidents, 1 of 13 Sessions, 1 of about 45 Runbooks.
- Not read: `tailscale-access-model`, `obsidian-vault-reorg-sep23`, Archive, `.obsidian`, `network-diagram.png`, most Incidents, Sessions and Runbooks.

## Confirmed by Daniel
- npm-dmz is LXC 2200 on [[pve-env1]] running Docker. [[ntfy]] is a container inside it, not a separate VM.
- [[ubuntu-401]] is on 10.0.30.40/24 (stated as "I believe", not verified on the host).
- Network+ is done, so the Active Directory lab is no longer blocked on it. It has not been started.

## Fixed
- [[ubuntu-401]]: `ip` was `VLAN30`, now 10.0.30.40. Added to the [[VLAN30]] host table.
- [[npm-dmz]]: `type`, `vmid` and `node` corrected to lxc, 2200, pve-env1. Body said "not Docker", corrected.
- [[ntfy]]: IP and node corrected, body now says it is a container inside npm-dmz. [[VLAN50]] table updated.
- [[pve-env1]] hosted list now includes npm-dmz and kalshi-mm. [[pve-services]] now lists [[pve-bastion]] and VLAN51.
- Bastion status was stale in Welcome, Network, Tailscale (which also said VLAN50), [[VLAN51]], [[pve-int-stepca]] and the Cloudflare SOP. Each now says the LXC is built and that the WARP Connector, sshd cert trust and SSH cutover were not recorded.
- Unresolvable links removed: `[[Hosts]]` in Infrastructure, `[[Projects/soc-stack-buildout]]` in SOC-Stack, and a self-link in Network.
- `[[soc-stack]]` was ambiguous between the [[soc-stack-vm]] host note (via alias) and the SOC-Stack MOC. All uses now point at `soc-stack-vm`.
- Markdown-style internal links converted to wikilinks in pve-bastion, VLAN11, VLAN30, VLAN51, PKI, SOC-Stack, Tailscale and the Cloudflare SOP.
- AD lab "deferred until after Network+" updated in Infrastructure and Security-Lab.
- Tailscale added to the MOC list in Conventions and to the overviews in Welcome. Known gaps in Conventions now match what Welcome claims.

## Open, needs a decision
- **WireGuard's role:** Network.md calls it primary and live. Tailscale.md calls it tertiary with a stale endpoint config. The heimdall note calls it one of three break-glass paths. Not changed.
- **DNS:** Network.md and Infrastructure.md still describe Pi-hole as primary for all VLANs and use `homelab.local`. [[dns-architecture]] (Sep 23) decided Unbound and `.internal`, and [[dhcp-dns-rollout]] is only planned. The MOCs do not say a change is pending.
- **ntfy `type`:** the Hosts/ schema has no value for a container, so it still says `vm`.
- **MOC duplication:** SOC-Stack repeats host IPs, VM IDs and VLAN10 firewall text, against the Conventions rule. PKI and Infrastructure have smaller repeats.
- **Legacy folders:** the 8 SOPs have no frontmatter. Runbook naming is mixed (`runbook-` prefix on some), `Runbooks/Cloudflare` is capitalized, and some Sessions and Incidents names carry no date. `runbook-docker-reset-container-init` and `runbook-linux-network-diagnostics-no-curl` sit in `Runbooks/proxmox/` but look like they belong elsewhere (judged from filenames only). Conventions defines no schema or naming for these folders.
- **Projects/:** Conventions cites the SSH bastion and step-ca migration as Project examples, but both live in SOPs.
- **kalshi-mm** has no Hosts note.
- [[pve-env1]] frontmatter `vlan` does not list VLAN50 although it now hosts npm-dmz.
- `network-diagram.png` is not embedded in any note read and may not show VLAN11 or VLAN51.

## Follow-up, later Sep 24
Daniel answered the open questions:
- **Remote access order:** bastion primary, WireGuard secondary (in progress), Tailscale tertiary and break-glass. Recorded in [[remote-access-path-order]]. Updated [[Network]], [[Tailscale]], [[heimdall]] and [[tailscale-access-model]] to match.
- **DNS:** "change pending" callouts added to [[Network]] and [[Infrastructure]], pointing at [[dns-architecture]] and [[dhcp-dns-rollout]].
- **Container type:** added `container` to the Hosts/ schema in [[Conventions]]. [[ntfy]] is now `type: container`.
- **Naming and frontmatter schema for SOPs, Runbooks, Incidents and Sessions:** agreed to build one. Daniel chose: rename existing files and fix links (after reading every note), ISO dates (`YYYY-MM-DD`) in Incident and Session names, and plain `<tool>-<task>` runbook names with no `runbook-` prefix. Rules are in [[Conventions]].

## Rename map (Sep 24, 2026)
This is the plan of record. To see what is already done, list the folders: a file is migrated when it has the new name.

**Runbooks/ (folder `Cloudflare` also becomes `cloudflare`):**
- authentik/: `runbook-authentik-diagnose-flow-failures` to `authentik-diagnose-flow-failures`, `runbook-authentik-oidc-external-service` to `authentik-oidc-external-service`. The other six `authentik-*` names are unchanged.
- cloudflare/: `runbook-cloudflare-tunnel-publish` to `cloudflare-tunnel-publish`, `runbook-warp-zero-trust-enrollment` to `cloudflare-warp-zero-trust-enrollment`.
- elk/: all six get an `elk-` prefix.
- npm/: all three get an `npm-` prefix.
- opnsense/: all get an `opnsense-` prefix, and `runbook-opnsense-isolated-vlan` becomes `opnsense-isolated-vlan`.
- pihole/: all three get a `pihole-` prefix.
- pki/: all get a `pki-` prefix, and `runbook-pki-check-cert-subject-formatting` and `runbook-stepca-import-existing-ca` lose `runbook-` (`pki-check-cert-subject-formatting`, `pki-stepca-import-existing-ca`).
- proxmox/: all get a `proxmox-` prefix, and every `runbook-` name loses it. `runbook-docker-reset-container-init` and `runbook-linux-network-diagnostics-no-curl` stay in proxmox/ because their own Category line says Proxmox (`proxmox-docker-reset-container-init`, `proxmox-linux-network-diagnostics-no-curl`).
- tailscale/: `runbook-tailscale-tailnet-policy` to `tailscale-tailnet-policy`.

**Incidents/ (date is the day the incident started):**
- `incident-review-jan25-workstation-routing` to `incident-2026-01-25-workstation-routing`
- `incident-review-feb23-corosync-stale-ips` to `incident-2026-02-23-corosync-stale-ips`
- `incident-review-march30` to `incident-2026-03-30-soc-dns-firewall-gap`
- `incident-review-may9` to `incident-2026-05-09-heimdall-log-pipeline`
- `incident-review-may22-shard-limit` to `incident-2026-05-19-elasticsearch-shard-limit`
- `incident-review-may24-power-onboot` to `incident-2026-05-24-power-outage-onboot`
- `incident-review-may27-pve-env1-silent-shutdown` to `incident-2026-05-27-pve-env1-silent-shutdown`
- `incident-review-jun2-pve-services-pmxcfs` to `incident-2026-05-30-pve-services-pmxcfs`
- `incident-review-sept14` to `incident-2026-09-13-elastic-log-shipping-outage`
- `incident-review-ntfy-authentik-outpost-fix-sept15-26` to `incident-2026-09-15-ntfy-authentik-outpost`
- `incident-review-vlan-failure-postmortem` to `incident-2026-01-10-vlan-implementation-failure`
- `incident-review-vlan-recovery` to `incident-2026-01-10-vlan-recovery`
- `incident-review-vlan-connectivity-fixes-jan2026` to `incident-2026-01-24-vlan-connectivity-fixes`
- `incident-review-vlan-security-lab-troubleshooting` to `incident-2026-01-24-security-lab-vlan-troubleshooting`

**Sessions/:**
- `session-proxmox-subnet-migration` to `session-2026-01-09-proxmox-subnet-migration`
- `session-log-march30` to `session-2026-03-30-soc-tls-wazuh-deployment`
- `session-logstash-tls-hardening` to `session-2026-05-02-logstash-tls-hardening`
- `session-log-sept13` to `session-2026-09-13-log-shipping-and-fleet-fixes`
- `session-log-sept14-authentik-devvlan` to `session-2026-09-14-authentik-devvlan`
- `session-log-sept15-cloudflare-hardening` to `session-2026-09-15-cloudflare-hardening`
- `session-log-sept18-bastion-path` to `session-2026-09-18-bastion-path`
- `session-log-sept19-stepca-iris-soar` to `session-2026-09-19-stepca-iris-soar`
- `session-log-sept20-bastion-mfa`, `-ca-migration`, `-documentation-rebuild` to `session-2026-09-20-<same slug>`
- `session-log-stepca-and-bastion-sept21` to `session-2026-09-21-stepca-and-bastion`
- `session-log-sept23-tailscale-policy` to `session-2026-09-23-tailscale-policy`
- `session-log-sept24-vault-audit` (this note) to `session-2026-09-24-vault-audit`

**Frontmatter added** to all 8 SOPs, all Runbooks, Incidents and Sessions, and the SOC buildout Projects notes.

## Migration result
- **Renames done and checked by listing:** 45 runbooks (all match their tool prefix), 14 incidents, 14 sessions. A filename search found no `runbook-`, `incident-review` or `session-log` names left.
- **Links fixed** in the runbooks, SOPs, Projects, Incidents, Sessions, VLAN11, Tailscale.md, the Cloudflare SOP and the bastion SOP. One dead link the first pass missed (a Kibana Lens runbook link at the end of `soc-phase1-baseline`) was found and fixed during verification.
- **Cleanups:** kalshi-mm removed from [[pve-env1]] (Daniel confirmed it is gone). [[ubuntu-401]] IP confirmed from two older notes. The `[[Hosts]]` link in [[obsidian-vault-reorg-sep23]] and the Welcome known-gaps line fixed.
- **Calls made without confirmation:** SOP statuses (`complete` for the Cloudflare tunnel, log source onboarding and VLAN implementation SOPs); `resolved` on the Jan 10 VLAN failure although the doc itself says unresolved, because the recovery doc resolves it; the mislabeled "Runbook:" prefix dropped from two Session headings; runbooks with a Proxmox Category kept in proxmox/. Change any of these freely.
- **Statuses from Daniel:** SOC phases 1 and 2 complete, phase 3 planned (untouched), phase 4 in progress (IRIS stood up), phase 5 planned (untouched).

## Not verified
- Link resolution was checked with the Link Integrity plugin on Sep 24 (19 broken links). Two were old-name links in [[dhcp-dns-rollout]], a note I had only read the first 30 lines of, now fixed. Seventeen were GitHub-style heading anchors that predate the migration (1 in [[sop-cloudflare-tunnel-implementation]], 16 in the [[sop-vlan-implementation]] table of contents), converted to Obsidian heading links. Re-run later Sep 24: 0 broken links, and 0 issues from the appendix heading links. The one isolated file was an empty stub, `set-vlan-tag-vm-network.md`, most likely created by clicking the dead link that used to be in [[proxmox-create-vm]]. Daniel moved it to Archive/.
- The middle sections of the largest legacy docs were not read (both January 10 VLAN incident docs, the January 24 security lab incident, `sop-vlan-implementation`, `session-2026-01-09-proxmox-subnet-migration`). An old link in there would be dead. The Link Integrity scan found no broken links in them apart from the SOP table of contents above.
- Five Sessions keep Markdown-style links where the link title contains an em dash. They resolve, but are not wikilinks.
- [[session-2026-09-18-bastion-path]] still records the earlier path order (Tailscale before WireGuard). Left as history; [[remote-access-path-order]] is current.

## Still open from the audit
- Resolved later Sep 24: [[pve-authtunnel]] now has a Hosts note (LXC 2100 on [[pve-services]], node found in a Sep 23 chat, existence confirmed by Daniel) and is in the [[VLAN50]] table. The `vlan` field on [[pve-env1]] and [[pve-services]] now lists the VLANs their guests use.
- Resolved later Sep 24: [[SOC-Stack]] no longer repeats host IPs, VM IDs or the VLAN10 firewall text. It links to [[VLAN10]] and the host notes instead. Nothing was lost, since both hold the full detail.
- Resolved later Sep 24: bastion state recorded as left in the Sep 21 session, confirmed by Daniel (see [[pve-bastion]], [[sop-ssh-bastion-build]], [[VLAN51]], [[remote-access-path-order]]). WARP enrollment blocked by `ERR_QUIC_PROTOCOL_ERROR`, sshd cert trust not configured, temporary port 22 rule not recorded as removed, no cutover. Still undecided: what the WireGuard path should be able to reach after the cutover.
- `network-diagram.png` was finalized Sep 20 ([[session-2026-09-20-documentation-rebuild]]), and Daniel confirmed on Sep 24 that it is largely accurate. Two known differences: ntfy is drawn as its own box but is a container inside `npm-dmz` (see [[ntfy]]), and [[pve-authtunnel]] (built Sep 21) is missing from VLAN50. Smaller differences come from later decisions: VLAN11 (planned Sep 23) is absent, VLAN60 is drawn as a NAS although nothing is deployed, and the remote access order set Sep 24 is not shown. It is embedded in [[Network]] with a caption listing the differences. The image can only be edited in draw.io.

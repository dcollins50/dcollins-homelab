---
name: Conventions
type: meta
status: live
---

Read this first in any session touching the vault. This is what keeps the structure consistent across sessions instead of drifting.

## Folder rules
- **Hosts/** — one note per physical node, VM, or LXC. Atomic: one machine, one file.
- **Network/** — one note per VLAN.
- **Runbooks/** — single-tool task procedures (e.g. "restart Wazuh agent," "renew a leaf cert"). Organized by tool subfolder (authentik/, opnsense/, pki/, proxmox/, etc.), including generic cross-cutting folders for patterns not tied to one product (linux/, ssh/), not just named software. Added Sep 25, 2026.
- **SOPs/** — multi-app processes with dependencies across tools (e.g. VLAN implementation, Cloudflare tunnel setup).
- **Decisions/** — one note per architectural or process decision, with the reasoning and what was ruled out. Not a status log — a record of *why*.
- **Projects/** — in-progress or completed multi-session builds (e.g. SSH bastion, step-ca migration). Links out to the Hosts/Network/Decisions notes it touches.
- **Incidents/** — postmortems. Added Sep 23, 2026 alongside the original six because it's a distinct recurring genre in this homelab (root cause, fix, what changed).
- **Sessions/** — dated raw session logs. Added Sep 23, 2026. Chronological record, not meant to be tidy — Decisions/ and Incidents/ are where the durable takeaways get written up. Multiple session files can share the same date — one file per distinct task/thread, not one file per calendar day. Clarified Sep 25, 2026.
- **Archive/** — superseded content kept for reference, not deleted (the filesystem connector currently has no delete capability, so this is also where anything retired lands).

Top-level MOC notes (Infrastructure.md, Network.md, PKI.md, Security-Lab.md, SOC-Stack.md, Tailscale.md) sit at the vault root, not in a folder. They hold conceptual/procedural material that doesn't belong to a single host, and link out to the atomic notes for specifics. Don't duplicate a host's IP/VLAN/status into a MOC — link to the Hosts/ note instead.

## Frontmatter schema

**Hosts/ notes:**
```yaml
---
name: <hostname>
type: physical-node | vm | lxc | container
vmid: <id or n/a>
node: <parent Proxmox node, or n/a for physical>
ip: <ip or subnet-range if unassigned>
vlan: <VLAN name/number>
status: live | planned | template
role: <one line>
---
```

**Network/ notes:**
```yaml
---
name: VLAN<n>
vlan_id: <n>
subnet: <cidr or TBD>
status: live | planned
role: <one line>
---
```

Keep frontmatter minimal. If a field doesn't apply, use `n/a` rather than omitting it, so Dataview queries don't break on missing keys.

A `container` note is for a Docker container that runs inside another host note's VM or LXC (for example [[ntfy]] inside [[npm-dmz]]). Use `vmid: n/a`, set `node` to the parent's Proxmox node, set `ip` to the parent's IP if it shares one, and name the parent in the body with a `[[wikilink]]`. Added Sep 24, 2026.

## Naming
- Host notes: lowercase-with-hyphens, matching the actual hostname (`pve-int-stepca.md`, not `stepca.md`).
- VLAN notes: `VLAN<n>.md`, capital VLAN.
- MOC notes: capitalized single word or hyphenated (`Infrastructure.md`, `SOC-Stack.md`).
- SOPs: `sop-<topic>.md`.
- Runbooks: `<tool>-<task>.md` inside a lowercase folder named for the tool (`opnsense/opnsense-add-firewall-rule.md`). No `runbook-` prefix. The tool prefix is what keeps the filename unique across the vault.
- Incidents: `incident-YYYY-MM-DD-<slug>.md`, dated by the day the incident started.
- Sessions: `session-YYYY-MM-DD-<slug>.md`.
- Decisions and Projects: `<topic>.md`, lowercase-with-hyphens.
- Every filename must be unique across the whole vault, because `[[wikilinks]]` resolve by filename. Adopted Sep 24, 2026.

## Frontmatter for other folders
Same rule as above: minimal, and `n/a` instead of omitting a field. `name` is the filename without `.md`.

**SOPs/ and Projects/:**
```yaml
---
name: <filename>
type: sop | project
status: planned | in-progress | complete
---
```

**Runbooks/:**
```yaml
---
name: <filename>
type: runbook
tool: <folder name>
---
```

**Incidents/:**
```yaml
---
name: <filename>
type: incident
date: YYYY-MM-DD
status: resolved | open
---
```

**Sessions/:**
```yaml
---
name: <filename>
type: session
date: YYYY-MM-DD
---
```

**Decisions/:** `name`, `type: decision`, `status: live | superseded`.

## Linking
Use `[[wikilinks]]` between any note that references a host, VLAN, or MOC that has its own file. Don't link to something that doesn't exist yet — create the stub or leave it as plain text until it does.

## Sourcing runbooks
Before concluding a procedure isn't documented, search past chat history, not just the vault. Real work often happens in chat, gets solved, and never gets written up, a runbook can often be reconstructed from an old session even when the vault has nothing. Added Sep 25, 2026, after discovering working Wazuh agent-enrollment and rootcheck-ignore procedures sitting unwritten in months-old chat sessions.

## Session workflow
Claude writes directly to the vault, no review inbox. At the start of a homelab session, read this file plus whatever Hosts/Network/Project notes are relevant to the work at hand — not the whole vault.

## Publishing
The vault is never edited to make something publishable. The public mirror in the repo's `docs/` folder is generated on demand with `Scripts/python/sanitize-doc.py`, which reads the vault and writes only to `docs/`. See [[doc-sanitization-policy]] and [[sanitize-publish-doc]]. Added Oct 1, 2026.

Any note can add `publish: false` to its frontmatter to stay out of the mirror. Absent means published. This is the one frontmatter key that is optional rather than `n/a`. The mirror updates automatically, see [[sanitize-automate-publish]]. Added Oct 2, 2026.

## Known gaps as of Sep 23, 2026
- No delete capability on the filesystem connector — retiring something means moving it to Archive/, not removing it.
- `docs/` folder is now empty (everything moved out) but can't be removed for the same reason.
- Runbooks/ and SOPs/ were migrated to the naming and frontmatter rules above on Sep 24, 2026 (see [[session-2026-09-24-vault-audit]]). Their prose has not been reviewed — they were moved as-is from the old repo docs.
- Incidents/ and Sessions/ were also migrated to the naming rules on Sep 24, 2026, but their prose is unreviewed. Three incident docs use pre-rename VLAN addressing and carry a note saying so: [[incident-2026-01-10-vlan-implementation-failure]], [[incident-2026-01-10-vlan-recovery]] and [[incident-2026-01-24-vlan-connectivity-fixes]].
- Some frontmatter `status` values were chosen during the Sep 24 migration and should be reviewed (the SOC phase docs 3 to 5, the SOP statuses). A Link Integrity scan on Sep 24 found no broken links beyond the ones fixed (see [[session-2026-09-24-vault-audit]]).

## Known gaps as of Sep 25, 2026
- A large batch of Runbooks/ (Wazuh, DFIR-IRIS, Uptime Kuma, Portainer, Tailscale, SSH, Vaultwarden, Gitea, Linux) were reconstructed from past chat sessions rather than the vault, per the new Sourcing runbooks rule above. These reflect what worked when the procedure was originally run in chat, not a live re-verification done at write time, treat them as accurate-as-documented rather than freshly confirmed.
- The chat-history sweep that produced those runbooks was not exhaustive. Explicitly not yet checked: Gitea beyond MFA, Vaultwarden backup (still not configured at all, a separate open item), the in-progress Shuffle workflow build itself, Ansible/Semaphore (nothing built yet), the Cloudflare Access/WARP side of the still-stalled SSH bastion, and general systemd/journalctl troubleshooting beyond what's in Runbooks/ssh/.

## Known gaps as of Oct 2, 2026
- [[SOC-Stack]] has no VM inventory and does not record that kalshi-mm was destroyed. Two notes pointed at a nonexistent `[[homelab-soc]]` for that content and were repointed to [[SOC-Stack]] as a stopgap: [[standardize-node-clocks-utc]] (line 38) and [[wazuh-enroll-agent]] (line 66). Their claims ("VM inventory list is the starting point", "kalshi-mm destroyed") are not backed by the note they now link to. Needs a vault note holding the VM inventory, checked against Proxmox before it is written, then the two links updated.

## Known gaps as of Oct 4, 2026
- Agentless Wazuh monitoring of [[opnsense]] is not working. The check logs in as root and root SSH is disabled (`permitrootlogin no`). Start date unknown. See [[session-2026-10-04-opnsense-25-7-11-patch]].
- [[opnsense-agentless-ssh-monitoring-setup]] said the passlist file is on OPNsense. It is on the Wazuh manager. Corrected Oct 4, 2026.
- The same runbook and [[opnsense]] imply OPNsense cannot run a native Wazuh agent. An official `os-wazuh-agent` plugin exists with limited community support. Compatibility with the 4.14.4 manager is unchecked.
- OPNsense is on the end-of-life 25.7 series. See [[opnsense-major-upgrade]].

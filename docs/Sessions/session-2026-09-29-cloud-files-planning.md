---
name: session-2026-09-29-cloud-files-planning
type: session
date: 2026-09-29
---

Planning session for [[nimbus]]. Nothing was built. Design is in [[cloud-files-design]], steps in [[sop-cloud-files-build]].

- Daniel wants a shared Google Drive style store with the collaborator. The same the collaborator who is on [[dev-git]].
- About 200GB. Tailscale only. No quota.
- Checked `pvesm status` on pve-env1: `local-lvm` 772GB free (thin, 7% used), `local` 83GB free, `pc-backup` (CIFS) shown as disabled. Daniel says `pc-backup` is for the Gitea instance only. No mention of it exists in the vault or in past chats, so it is not documented here. Add it if it turns out to matter.
- Checked D: on the workstation: 887GB free, 976GB used.
- Backups: Daniel chose a scheduled backup to D: on the workstation. Settings and database on the dev-git pattern, files as a single mirror.
- Found in the docs: Tailscale HTTPS certificates publish machine and tailnet names publicly; Nextcloud's docs list Debian 12, not 13; Nextcloud 35.0.0 is two weeks old. All three are open decisions in the SOP.

## Sep 30, 2026 build progress
- Container `nimbus` (CT 3400) built, locked down and joined to the tailnet. Details in [[nimbus]].
- Hostname `nimbus` was Daniel's deliberate choice. Vault notes were renamed to match.
- Wizard requires a root password or SSH key, so a dedicated `nimbus` key was used. My first advice to clear it was wrong.
- `pct config` showed a stray `unused0` volume after the `-mp0` command ran twice. Removed with `pct set 3400 --delete unused0`.
- Tailnet policy updated and saved with the new grants and tests.
- Still open: Nextcloud version (35.0.0 vs previous major 34), HTTPS decision (plain HTTP inside Tailscale for now), whether key expiry was disabled on `nimbus`.

The rest of the build, HTTPS, accounts and backups are in [[session-2026-09-30-cloud-files-build]].

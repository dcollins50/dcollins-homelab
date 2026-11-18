---
name: cloud-files-design
type: decision
status: live
---

## Decision
Decided Sep 29, 2026. Daniel and the collaborator get a self-hosted, Google Drive or OneDrive style file store: Nextcloud in its own LXC, [[nimbus]], on [[pve-env1]] in the DMZ ([[VLAN50]]). Reached only over Tailscale machine sharing, the same pattern as [[dev-git]].

## What Daniel asked for
- Feels like Google Drive or OneDrive: web interface, sync apps, sharing between the two users.
- About 200GB of general files, whatever is needed.
- Tailscale only, "since that's where we're at".
- No storage quota. Daniel and the collaborator will deal with space between themselves if it becomes a problem.

## Choices
- **Nextcloud, not File Browser or Syncthing.** File Browser is a web page only with no sync apps. Syncthing syncs folders but has no web interface. Nextcloud is the only one of the three that gives the Drive or OneDrive experience.
- **Own container, not shared with dev-git.** Same reasoning as [[gitea-collab-access]]: the collaborator's access should not sit on the box that holds the git projects.
- **Native install with PostgreSQL.** Daniel prefers LXCs over Docker for single-application services. Nextcloud's docs recommend MySQL or MariaDB and say SQLite is only for testing and minimal instances, so a real database is used. PostgreSQL chosen Sep 30, 2026: Nextcloud's system requirements list PostgreSQL 13 to 17, while MariaDB is listed only up to 11.4 and Debian 13 ships a newer MariaDB. Confirm the installed PostgreSQL version with `apt policy postgresql` at build time.
- **Data disk on `local-lvm`.** 772GB free there. The 238GB secondary NVMe was rejected: 200GB of data would leave no room to grow.
- **No new OPNsense rules.** The DMZ already blocks all traffic into internal VLANs and allows outbound internet, which is all Tailscale needs.

## Backups
Decided Sep 29, 2026, built Sep 30, 2026. Two separate jobs, because the Proxmox `local` storage only has about 83GB free and cannot hold a 250GB disk:
1. **Container settings, accounts and database:** weekly Proxmox backup to `local`, keep 3, data disk excluded (`backup=0`). Same pattern as [[dev-git-backup-strategy]].
2. **The files themselves:** one mirror of the data on the workstation's BitLocker D: drive, not three copies. D: had 887GB free on Sep 29, 2026, so a 250GB mirror fits.

The planned method for the file mirror (rclone over WebDAV with an app password) was replaced by a weekly read-only snapshot of the data disk on [[pve-env1]] pulled with rsync through a restricted key, with changed and deleted files kept 28 days. The as-built design, the limits and the open risks are in [[nimbus-backup-design]]. A plain Nextcloud sync client is not a backup, because deletions would sync too.

Still true from the plan:
- The first copy is slow. The dev-git backup ran at about 6MB per second over Tailscale's relay, so 200GB is roughly 9 hours. Only changes copy after that.
- Both pulls run only while the workstation is on, Tailscale is running and D: is unlocked.
- Proxmox snapshot backups of a running database are crash-consistent, not clean. A scheduled database dump inside the container is an open item.

## Decided Sep 30, 2026
- Nextcloud 34 (the previous major), not 35.0.0. Nextcloud's download server has `latest-34.tar.bz2` with `.asc` and `.sha256` files beside it. Debian 13's PHP 8.4 is on the supported PHP list.
- **HTTPS through `tailscale serve`.** Daniel chose real HTTPS. Plain HTTP was used first while building, then switched. The cost is that Tailscale's certificate feature publishes the tailnet DNS name and machine names in a public certificate log, which Daniel accepted. the collaborator can reach the `tailscale serve` endpoint through the machine share (tested, works). The container firewall now allows only port 443.
- **Two-factor enforced for all users** (TOTP, in Administration settings, Security).
- **Accounts:** `collab-user` and `[admin-user]` are regular users. The admin account `[nimbus-admin]` was disabled the same day, after `[admin-user]` logged in with two-factor. Re-enable with `occ user:enable [nimbus-admin]` when admin work is needed, then disable it again.
- **Mirror with a 28-day changed and deleted history** is the default built for the file backup. Not explicitly confirmed by Daniel against a full-history alternative.

## Considered, not done: single sign-on
Authentik through Nextcloud's `user_oidc` app. For two people it adds dependencies without a matching benefit, and two constraints in this build get in the way: the DMZ cannot reach RFC1918 addresses, and the collaborator's browser cannot reach [[authentik]]. Daniel's counterpoint is that SSO earns its place for an organization with users around the world. That is a separate deployment question, not decided. Cloudflare Zero Trust versus Tailscale as the access layer for such a deployment is also open.

## Open
- Nextcloud desktop and phone client test over HTTPS.
- Open items on the host are listed in [[nimbus]].

## Related
- [[nimbus]]
- [[sop-cloud-files-build]]
- [[nimbus-backup-design]]
- [[dev-git-backup-strategy]]
- [[tailscale-access-model]]
- [[gitea-collab-access]]

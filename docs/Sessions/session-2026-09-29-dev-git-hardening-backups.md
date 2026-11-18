---
name: session-2026-09-29-dev-git-hardening-backups
type: session
date: 2026-09-29
---

Hardened [[dev-git]] and set up its backups. Full commands in [[sop-gitea-collab-build]] steps 11 and 12.

- Friend reached the web UI. Cause of his earlier DNS_PROBE_POSSIBLE not recorded.
- Daniel's decisions: new repos private by default; container SSH off, manage with `pct enter` only; Wazuh agent yes, with a scoped OPNsense rule; backups weekly, keep 3, to pve-env1 and the workstation's BitLocker D:.
- Found before hardening: sshd with password login on; Gitea and sshd listening on the DMZ address as well as Tailscale; nftables wide open; OpenID sign-in and sign-up on. Auto updates were already on.
- Applied: Gitea settings (OpenID off, emails private, repos private by default); sshd disabled and masked; nftables `devgit` table (Tailscale-only access to 2222 and 3000, no `flush ruleset`); Tailscale policy with port 22 removed and a deny test added. Reboot test passed.
- Tailscale policy save first failed with `address "dev-git:22" ... want: Accept, got: Drop`. That was expected: the old test still said 22 should be open. Fixed by moving `dev-git:22` to the deny list.
- `tailscale status` on dev-git lists all of Daniel's machines. That's from the Tailscale SSH rule (`autogroup:member` to `autogroup:self`), not access: the grants and the dev-git deny test decide what's reachable.
- Proxmox backup job created. The schedule runs in UTC, so `sun 03:00` is Saturday 10pm Central. First run OK, about 382MB.
- The backup key was first generated on pve-env1 by mistake (PowerShell syntax in bash), creating `:USERPROFILE.sshdevgit_backup_pull` in admin's home. Deleted, regenerated on the workstation.
- Forced-command gatekeeper on pve-env1 verified (`whoami` and `get /etc/shadow` denied). Manual pull worked (381663418 bytes, size matched). The scheduled task run returned "up to date". An earlier 19:29 error said D: wasn't available, and D: was locked at that moment.
- Vault edits from Sep 28 and earlier on Sep 29 turned up reverted on the workstation copy (SOP, dev-git host note, Sep 28 session). Most likely cause: the tool Claude uses to write to the vault resent an old copy when the same upload path was reused, not Obsidian. Rewritten from fresh paths, and file sizes on disk checked afterward.

## Still open
- Wazuh agent: OPNsense `dev_git` alias and DMZ pass rule, then enrollment.
- Workstation reaches dev-git through Tailscale's relay, not directly. Possibly OPNsense outbound NAT. Not investigated.
- Delete the leftover `Hosts/gitea-collab.md`.

## Related
- [[dev-git]]
- [[dev-git-backup-strategy]]
- [[session-2026-09-28-dev-git-build]]

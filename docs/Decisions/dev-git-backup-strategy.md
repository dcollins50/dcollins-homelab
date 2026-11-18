---
name: dev-git-backup-strategy
type: decision
status: live
---

## Decision
Decided by Daniel, Sep 29, 2026. [[dev-git]] (CT 3300) is backed up weekly, not nightly, keeping the newest 3 and overwriting the oldest. Two copies:
- Proxmox backup job on [[pve-env1]] to its `local` storage (the only storage there that accepts backups), `sun 03:00` UTC, keep last 3.
- The workstation pulls the newest backup to its BitLocker D: drive, keeping the last 3 there too because space is limited. Runs as a Windows scheduled task on Sundays at 1pm and at every logon.

## Why
- dev-git holds real projects. Every git clone is already a full copy of the code and history, so what only Gitea has is issues, pull requests, accounts and settings. Those need a backup.
- The storage VLAN and NAS aren't built (budget), so a copy on pve-env1 alone would live on the same machine as dev-git. The D: drive is the off-machine copy.
- **The workstation pulls; dev-git never pushes.** dev-git is in the DMZ and the tailnet policy test forbids it from reaching any other machine. Letting it push to the workstation would undo that.
- **The pull key is locked to one script.** (Updated Sep 30, 2026: the same script and key also serve [[nimbus]] container backups, see [[nimbus-backup-design]]. The script now answers for `vzdump-lxc-3300-*` and `vzdump-lxc-3400-*` files only.) A scheduled task can't enter a passphrase, so the key has none. On pve-env1 it's restricted with `command="/usr/local/bin/devgit-backup-send",restrict`, so a leaked key can only fetch dev-git backup files: no shell, no other commands, no forwarding. Verified that `whoami` and `get /etc/shadow` are refused.
- The backups contain the whole container, including Gitea's secret keys, so they're kept only on encrypted storage.

## Trade-offs accepted
- The key sits in the `admin` account on pve-env1, which has sudo, and relies on the forced command for safety. A dedicated account with no sudo would be cleaner. Not done.
- The pull only works while the workstation's Tailscale is running and D: is unlocked. Failures are logged to `%LOCALAPPDATA%\devgit-backup-pull.log` on C:.

## Related
- [[nimbus-backup-design]]
- [[sop-gitea-collab-build]] (step 12 has the commands)
- [[dev-git]]
- [[pve-env1]]

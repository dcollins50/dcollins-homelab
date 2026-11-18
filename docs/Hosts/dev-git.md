---
name: dev-git
type: lxc
vmid: 3300
node: pve-env1
ip: 10.0.50.x
vlan: VLAN50
status: live
role: Separate Gitea for collaborating with an outside user, shared over Tailscale
---

Built Sep 28, 2026. VMID 3300. IP 10.0.50.x.

Debian 13 unprivileged LXC, 2 cores / 1GB RAM / 8GB disk (sized to match [[npm-dmz]], an estimate rather than an official Gitea figure). Gitea installed natively from the official binary, SQLite database. Web UI on port 3000 and Gitea's built-in SSH server on port 2222, both reached only over Tailscale. No Cloudflare hostname, and no OPNsense rules beyond the existing DMZ policy.

## Current state (Sep 28, 2026)
- Tailscale address 100.x.x.x. Tailnet name `[tailnet].ts.net`, so the full name is `dev-git.[tailnet].ts.net`. Key expiry disabled on this node.
- Gitea 1.27.3 installed natively, running as a systemd service, first-run setup done with SQLite. Web UI at `http://dev-git.[tailnet].ts.net:3000`.
- Container was built without `/dev/net/tun` at first; fixed with `pct set 3300 --dev0 /dev/net/tun` and a reboot.
- Tailnet policy updated with the dev-git alias, the two grants, and the dev-git deny test.
- Shared to the friend's Tailscale account. He got in on Sep 29, see [[session-2026-09-28-dev-git-build]].
- Sep 29, 2026: app.ini locked down (SSH port typo 22222 fixed to 2222, built-in SSH server on 2222, sign-up closed, sign-in required to view, two-factor enforced, org creation off). Backup at `/etc/gitea/app.ini.bak`. Outside user reached the web UI. Daniel's SSH works with a dedicated `dev_git` key.
- Sep 29, 2026 hardening: container SSH disabled and masked (manage with `pct enter 3300`), nftables `devgit` table drops everything except Tailscale traffic to 2222 and 3000, OpenID sign-in and sign-up off, email private by default, new repos private by default. Tailscale policy no longer allows port 22.
- Backups: weekly Proxmox backup on pve-env1 (`local`, keep 3, `sun 03:00` UTC), pulled to the workstation's D: drive (keep 3) by a scheduled task. See [[dev-git-backup-strategy]].
- Not done yet: Wazuh agent (OPNsense rule plus enrollment), the rest of the verify steps in the SOP.

Deliberately separate from the internal Gitea on [[services-host]]. See [[gitea-collab-access]] for why.

## Build
[[sop-gitea-collab-build]]

## Related
- [[pve-env1]]
- [[VLAN50]]
- [[gitea-collab-access]]
- [[Tailscale]]

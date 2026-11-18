---
name: session-2026-09-28-dev-git-build
type: session
date: 2026-09-28
---

Built [[dev-git]], a separate Gitea for collaborating with a friend, shared to him over Tailscale. Decision in [[gitea-collab-access]], build steps in [[sop-gitea-collab-build]].

## Done
- Checked node memory before placement: pve-env1 56%, pve-env2 82%, pve-services 40%, pve-gateway 19%. Built on [[pve-env1]].
- LXC 3300, hostname dev-git, VLAN50, 10.0.50.x. Named by Daniel (proposed name was gitea-collab, proposed ID was 2203).
- `/dev/net/tun` was missing inside the container. Fixed with `pct set 3300 --dev0 /dev/net/tun` plus a reboot.
- Tailscale joined: 100.x.x.x, tailnet `[tailnet].ts.net`. Key expiry disabled on dev-git (Daniel's call, so the outside user doesn't lose access silently).
- Tailnet policy: live policy matched [[tailscale-tailnet-policy]] exactly, replaced with a version adding the dev-git alias, a workstation grant (22, 3000, 2222), an `autogroup:shared` grant (3000, 2222) and a dev-git deny test. Saved without errors (not explicitly confirmed, assumed from Daniel continuing).
- Gitea 1.27.3 downloaded, signature checked, installed natively, systemd service running. The `pct enter` shell's PATH lacked `/usr/local/bin`, harmless. A perl locale warning during `adduser` was harmless.
- First-run setup done in the browser: SQLite3, SSH port 2222, base URL on the `.ts.net` name.
- The workstation couldn't resolve `dev-git.[tailnet].ts.net` until the Tailscale app was started on it.
- Machine shared to the friend's Tailscale account.

## Friend can't connect (open)
- First invite apparently landed on a different account: his `tailscale status` showed only his own machine (matrix1, account [collaborator-tailscale-account]). Invite re-sent.
- His `nslookup` against 100.100.100.100 timed out, so his Tailscale DNS wasn't answering. MagicDNS or "Use Tailscale DNS settings" may be off on his side.
- After that: `tailscale ping` to dev-git worked, but `Test-NetConnection 100.x.x.x -Port 3000` failed (ping status 11050) and the browser showed ERR_NETWORK_ACCESS_DENIED.
- `journalctl -u tailscaled -f | grep -i drop` on dev-git showed nothing during his test, so his traffic never reached dev-git. The dev-git policy and host are not the cause. (tcpdump on `tailscale0` was not a valid test for this: Tailscale's policy filter runs before packets reach that interface.)
- His machine: only Windows Defender, Windows Firewall outbound NotConfigured, and a NordVPN adapter (NordLynx) present. Leading suspect: NordVPN's kill switch refusing non-NordVPN traffic. Possible Meshnet clash, since it uses the same 100.64.0.0/10 range as Tailscale.
- Next test: fully quit NordVPN and re-run `Test-NetConnection`. If that doesn't do it: test from his phone on Tailscale, reboot without NordVPN autostarting, check Windows Firewall outbound block rules, check `Find-NetRoute`, check his Tailscale client version.

## Not done yet
- app.ini lockdown (step 8 of the SOP). Next command: `grep -nE '^\[|START_SSH|SSH_PORT|SSH_LISTEN|DISABLE_REG|REQUIRE_SIGNIN|TWO_FACTOR|INSTALL_LOCK|ORGANIZATION' /etc/gitea/app.ini`
- Confirm his Gitea account and repo collaborator (Write) were created.
- Verify steps (step 10 of the SOP).
- Daniel asked to "scope a rule for dev-git" but didn't say which one. Options were listed (Wazuh agent rule, LAN access, VLAN20 access, a Tailscale policy change, tightening outbound). Not decided.
- Delete the leftover `Hosts/gitea-collab.md` pointer file.

## Sep 29, 2026 follow-up
- Considered dropping Tailscale for a public Cloudflare hostname. Blocker found: Cloudflare's free plan caps each request at 100MB, and a git push over HTTPS is one request. Workaround identified (git over SSH through the tunnel with `cloudflared` on the client, LFS over SSH via `LFS_ALLOW_PURE_SSH`), but stayed on Tailscale.
- Friend confirmed no VPNs running, Tailscale only, still got DNS_PROBE_POSSIBLE, then got in. What fixed it was not recorded yet.
- app.ini check found the setup page hadn't saved as intended: `SSH_PORT = 22222` (typo), `DISABLE_REGISTRATION = false`, `REQUIRE_SIGNIN_VIEW = false`. Fixed along with the rest of step 8 of [[sop-gitea-collab-build]]. Backup kept at `/etc/gitea/app.ini.bak`.
- Daniel decided not to use his homelab admin key on dev-git. Made a dedicated `dev_git` key and added a matching entry to his Windows SSH config. `ssh -T` authenticated as the Gitea user `admin`.
- Last night's SOP edits were missing from the vault copy on the workstation (possibly overwritten by Obsidian). Rewritten.

## Related
- [[dev-git]]
- [[sop-gitea-collab-build]]
- [[gitea-collab-access]]

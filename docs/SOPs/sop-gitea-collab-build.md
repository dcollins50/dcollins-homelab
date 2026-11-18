---
name: sop-gitea-collab-build
type: sop
status: in-progress
---

# SOP: Build dev-git and Share It Over Tailscale

**Objective:** Stand up a separate Gitea ([[dev-git]]) in the DMZ and give one outside user access to it through Tailscale machine sharing, limited to Gitea's web and SSH ports. Reasoning is in [[gitea-collab-access]].

**Progress (Sep 29, 2026):** steps 1 to 8 done, container hardening (step 11) and backups (step 12) done. Outside user reached the web UI on Sep 29, SSH on 2222 confirmed from the workstation with a dedicated key. Open: Wazuh agent (OPNsense rule plus enrollment), the rest of step 10. See [[session-2026-09-28-dev-git-build]] and [[session-2026-09-29-dev-git-hardening-backups]].

**Written:** Sep 28, 2026, before the build. Commands were checked against Gitea's and Tailscale's current documentation that day, but nothing here has been run yet. Update this note with whatever actually happened.

**Placeholders used below:**
- `<tailnet>` is your tailnet's MagicDNS name (shown on the DNS page of the Tailscale admin console). The full hostname becomes `dev-git.<tailnet>.ts.net`.
- `<dev-git-ts-ip>` is the 100.x address Tailscale gives the container.

---

## 1. Create the container (Proxmox UI, pve-env1)

Before starting, confirm that 10.0.50.x is unused. Check it with `ping 10.0.50.x` from [[npm-dmz]].

| Setting | Value |
|---|---|
| Node | pve-env1 |
| CT ID | 3300 |
| Hostname | dev-git |
| Unprivileged | Yes |
| Template | Debian 13 |
| Disk | 8GB |
| Cores | 2 |
| Memory / Swap | 1024 MB / 512 MB |
| Network | Same bridge as npm-dmz, VLAN tag 50, static `10.0.50.x/24`, gateway `10.0.50.x` |
| DNS | `10.0.50.x` (not Pi-hole; the DMZ blocks RFC1918, which is what made apt hang on npm-dmz, see [[sop-cloudflare-tunnel-implementation]]) |

Don't start it yet. First give it the tunnel device Tailscale needs. (Missed on the first build on Sep 28, 2026: `/dev/net/tun` didn't exist inside the container until this was run and the container rebooted.) On the pve-env1 shell:
```bash
pct set 3300 --dev0 /dev/net/tun
pct start 3300
```

## 2. Base packages (inside the container)
```bash
apt update && apt full-upgrade -y
apt install -y git curl gnupg unattended-upgrades
```

## 3. Join the tailnet
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```
Open the login URL it prints and approve the machine under your account. Then run:
```bash
tailscale ip -4
```
and write down the 100.x address.

**Your own device needs Tailscale running** to reach dev-git by name. On Sep 28, 2026 the workstation got a DNS error for the `.ts.net` name until the Tailscale app was started. The same applies to the outside user.

If `tailscale up` hangs with no output, run through the Pi-hole sinkhole check in [[tailscale-tailnet-policy]] first.

**Key expiry:** by default the node key expires and has to be re-authenticated, which cut off every node in June 2026 without anyone noticing. With the default left on, your friend loses access silently when the key expires. Decide in the Machines page whether to disable expiry on this one node.

## 4. Update the tailnet policy (Tailscale admin console, Access controls)

Do this from the LAN, same as the original policy apply. Add the host alias:
```
"dev-git": "<dev-git-ts-ip>",
```
Add two grants:
```
{ "src": ["workstation"],
  "dst": ["dev-git"],
  "ip":  ["tcp:22", "tcp:3000", "tcp:2222"] },
{ "src": ["autogroup:shared"],
  "dst": ["dev-git"],
  "ip":  ["tcp:3000", "tcp:2222"] },
```
Add a test proving the new box can't reach anything:
```
{ "src": "dev-git",
  "deny": ["pve-env1:22", "pve-env1:8006", "services-host:22", "heimdall:22", "workstation:22"] },
```
Save. If any test fails, Tailscale refuses the save and names the failing test.

Port 22 is only for your own admin SSH into the container. Drop it from the workstation grant if you'd rather manage it with `pct enter 3300` from pve-env1.

## 5. Install Gitea (inside the container)

Check https://dl.gitea.com/gitea/ for the current version. It was 1.27.3 in the docs on Sep 28, 2026.
```bash
cd /tmp
wget -O gitea https://dl.gitea.com/gitea/1.27.3/gitea-1.27.3-linux-amd64
wget https://dl.gitea.com/gitea/1.27.3/gitea-1.27.3-linux-amd64.asc
gpg --keyserver hkps://keys.openpgp.org --recv 7C9E68152594688862D62AF62D9AE806EC1592E2
gpg --verify gitea-1.27.3-linux-amd64.asc gitea
```
Look for `Good signature`. Then:
```bash
chmod +x gitea
cp gitea /usr/local/bin/gitea

adduser --system --shell /bin/bash --gecos 'Git Version Control' \
  --group --disabled-password --home /home/git git

mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/
mkdir /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```
`/etc/gitea` stays writable by the git user only until the web installer writes `app.ini`. Step 8 locks it back down.

## 6. systemd service

Based on Gitea's sample unit (`contrib/service/systemd/gitea.service`), trimmed to what applies here: SQLite, so no database dependency. Create `/etc/systemd/system/gitea.service`:
```ini
[Unit]
Description=Gitea (Git with a cup of tea)
After=network.target

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
RestartSec=2s
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload
systemctl enable gitea --now
systemctl status gitea
```

## 7. Web installer (from the workstation, over Tailscale)

Open `http://dev-git.<tailnet>.ts.net:3000`. It's plain HTTP, but the traffic is inside Tailscale's encrypted tunnel.

- Database: SQLite3
- Server Domain and SSH Server Domain: `dev-git.<tailnet>.ts.net`
- SSH Server Port: `2222`
- HTTP Listen Port: `3000`
- Base URL: `http://dev-git.<tailnet>.ts.net:3000/`
- Optional settings: disable self-registration, require sign-in to view pages
- Administrator account: create **your** admin account here, so the first account on the box is never someone else's

## 8. Harden app.ini (inside the container)

**Check before editing, don't assume the setup page saved things.** On Sep 29, 2026 the installed `app.ini` had `SSH_PORT = 22222` (a typo on the setup page) and both `DISABLE_REGISTRATION` and `REQUIRE_SIGNIN_VIEW` still `false`, even though the plan was to tick those boxes. Run this first to see the real state:
```bash
grep -nE '^\[|START_SSH|SSH_PORT|SSH_LISTEN|DISABLE_REG|REQUIRE_SIGNIN|TWO_FACTOR|INSTALL_LOCK|ORGANIZATION' /etc/gitea/app.ini
```
What was actually run on Sep 29 (back up first, then targeted edits):
```bash
cp -p /etc/gitea/app.ini /etc/gitea/app.ini.bak
sed -i \
  -e 's/^SSH_PORT = 22222$/SSH_PORT = 2222/' \
  -e 's/^DISABLE_REGISTRATION = false$/DISABLE_REGISTRATION = true/' \
  -e 's/^REQUIRE_SIGNIN_VIEW = false$/REQUIRE_SIGNIN_VIEW = true/' \
  -e 's/^DEFAULT_ALLOW_CREATE_ORGANIZATION = true$/DEFAULT_ALLOW_CREATE_ORGANIZATION = false/' \
  -e '/^\[server\]$/a START_SSH_SERVER = true' \
  -e '/^\[server\]$/a SSH_LISTEN_PORT = 2222' \
  -e '/^INSTALL_LOCK = true$/a TWO_FACTOR_AUTH = enforced' \
  /etc/gitea/app.ini
printf '\n[admin]\nDISABLE_REGULAR_ORG_CREATION = true\n' >> /etc/gitea/app.ini
chmod 750 /etc/gitea
chmod 640 /etc/gitea/app.ini /etc/gitea/app.ini.bak
systemctl restart gitea
```
The target end state is below.

Edit `/etc/gitea/app.ini` so these sections include these lines. Add any that are missing and leave the rest of the file alone:
```ini
[server]
START_SSH_SERVER = true
SSH_PORT = 2222
SSH_LISTEN_PORT = 2222

[service]
DISABLE_REGISTRATION = true
REQUIRE_SIGNIN_VIEW = true
DEFAULT_ALLOW_CREATE_ORGANIZATION = false

[admin]
DISABLE_REGULAR_ORG_CREATION = true

[security]
INSTALL_LOCK = true
TWO_FACTOR_AUTH = enforced
```
`TWO_FACTOR_AUTH = enforced` needs Gitea 1.24 or newer. Like [[gitea-enable-mfa]] warns for the internal instance, it covers the web login only. SSH keys and access tokens don't go through it.

Lock the config back down and restart:
```bash
chmod 750 /etc/gitea
chmod 640 /etc/gitea/app.ini
systemctl restart gitea
```
Log back in. You should be made to set up TOTP. Save the recovery codes outside Gitea and outside Vaultwarden.

## 8b. Dedicated SSH key per person

Don't reuse an existing key (the homelab admin key or the VPS key) for dev-git. Each person makes a key used only here. On Windows:
```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\dev_git -C "dev-git"
Add-Content -Path $env:USERPROFILE\.ssh\config -Value @"

Host dev-git.<tailnet>.ts.net
    User git
    Port 2222
    IdentityFile ~/.ssh/dev_git
    IdentitiesOnly yes
"@
Get-Content $env:USERPROFILE\.ssh\dev_git.pub | Set-Clipboard
```
Paste into Gitea under Settings, SSH / GPG Keys. Test with `ssh -T dev-git.<tailnet>.ts.net`, which should greet you by Gitea username and say Gitea doesn't provide shell access. `IdentitiesOnly yes` stops SSH from also offering other keys to dev-git. Done for Daniel on Sep 29, 2026.

## 9. Set up his access

In Gitea:
1. Site Administration, User Accounts, Create User Account. Regular user, not admin.
2. Create the project repo under your account.
3. Repo Settings, Collaborators, add him with **Write**, not Admin.

In Tailscale:

4. Machines page, the `...` menu on dev-git, Share. Send him the invite by email or link.
5. He accepts it from his own tailnet, where he has to be an Owner or Admin. A free personal tailnet is fine.

He then:

6. Logs in at `http://dev-git.<tailnet>.ts.net:3000`, changes his password and sets up TOTP (enforced).
7. Adds his own SSH public key under Settings, SSH / GPG Keys.
8. Clones with `ssh://git@dev-git.<tailnet>.ts.net:2222/<your-user>/<repo>.git`.

**If his browser shows DNS_PROBE_POSSIBLE:** his machine can't look up the `.ts.net` name. It's never Cloudflare or OPNsense on this path. Check, in order: dev-git appears in his `tailscale status` (if not, the invite was accepted on a different Tailscale account); `Test-NetConnection 100.x.x.x -Port 3000` succeeds; MagicDNS on in his tailnet's DNS page and "Use Tailscale DNS settings" checked in his client. Fallback: a hosts file line `100.x.x.x dev-git.<tailnet>.ts.net`. On Sep 28 a NordVPN install on his machine was also refusing connections before they left it (ERR_NETWORK_ACCESS_DENIED). Proof it's his side: `journalctl -u tailscaled -f | grep -i drop` on dev-git shows nothing while he tests (Tailscale logs policy drops there; tcpdump on `tailscale0` can't prove this because the policy filter runs before that interface).

## 10. Verify

- **He can:** load the web UI, and clone and push over SSH on 2222.
- **He can't reach port 22** on dev-git: `nc -zv dev-git.<tailnet>.ts.net 22` from his machine should fail.
- **The box can't reach the tailnet:** from inside dev-git, `nc -zv -w3 100.x.x.x 22` (pve-env1's tailnet address) should fail.
- **The box can't reach the LAN:** from inside dev-git, `nc -zv -w3 10.0.20.x 22` should fail. That's the existing DMZ block.
- **Tailscale** saved the policy with the new test in place.


## 11. Harden the container (done Sep 29, 2026)

State found before hardening: container sshd listening on all interfaces with password login on and root key login allowed; Gitea's 3000 and 2222 also listening on all interfaces, so reachable from the rest of the DMZ (including [[npm-dmz]]), not just Tailscale; the nftables input chain wide open; OpenID sign-in and sign-up on. Automatic security updates were already on (`20auto-upgrades` set, `unattended-upgrades` and `apt-daily-upgrade.timer` enabled).

**Gitea settings** (backups `app.ini.bak2`, `app.ini.bak3` in `/etc/gitea`):
```bash
sed -i \
  -e 's/^ENABLE_OPENID_SIGNIN = true$/ENABLE_OPENID_SIGNIN = false/' \
  -e 's/^ENABLE_OPENID_SIGNUP = true$/ENABLE_OPENID_SIGNUP = false/' \
  -e 's/^DEFAULT_KEEP_EMAIL_PRIVATE = false$/DEFAULT_KEEP_EMAIL_PRIVATE = true/' \
  /etc/gitea/app.ini
sed -i '/^\[repository\]$/a DEFAULT_PRIVATE = private' /etc/gitea/app.ini
systemctl restart gitea
```

**Container SSH off for good.** Managed only with `pct enter 3300` from pve-env1 (Daniel's decision, Sep 29, 2026). Gitea's own SSH on 2222 is separate and stays.
```bash
systemctl disable --now ssh.service ssh.socket
systemctl mask ssh.service ssh.socket
```
The Tailscale policy was updated to match: `tcp:22` removed from the workstation to dev-git grant, `dev-git:22` moved from the workstation test's accept list to its deny list.

**Firewall: only Tailscale traffic gets in.** `/etc/nftables.conf`, enabled with `systemctl enable nftables`. Deliberately has no `flush ruleset` line (the Debian stock file does), because that would wipe Tailscale's own iptables-nft tables on every nftables restart. It only replaces its own `devgit` table.
```
#!/usr/sbin/nft -f
table inet devgit
delete table inet devgit

table inet devgit {
    chain input {
        type filter hook input priority filter; policy drop;
        iif "lo" accept
        ct state established,related accept
        ct state invalid drop
        udp dport 41641 accept
        iifname "tailscale0" tcp dport { 2222, 3000 } accept
        ip protocol icmp accept
        meta l4proto ipv6-icmp accept
    }
}
```
Always check before loading: `nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf`. Survived a container reboot with Tailscale still connected.

**Checks run:** `grep -n -A1 '^\[repository\]$' /etc/gitea/app.ini` shows `DEFAULT_PRIVATE = private`; `ss -ltn` shows only 2222 and 3000.

**Postfix** listens on localhost:25 only. Left in place, low risk.

## 12. Backups (done Sep 29, 2026)

Weekly, keep 3, on pve-env1 and on the workstation's BitLocker D: drive. Reasoning in [[dev-git-backup-strategy]].

**Proxmox backup job** (Datacenter, Backup): node pve-env1, storage `local`, schedule `sun 03:00`, only CT 3300, ZSTD, Snapshot mode, retention Keep Last 3 (Keep all unchecked). **The schedule is in UTC** (nodes run UTC, see [[standardize-node-clocks-utc]]), so it runs Saturday 10pm Central. First run (Run now, Sep 29) finished `TASK OK`, about 382MB, snapshot mode worked with the `/dev/net/tun` passthrough.

**Workstation pull.** dev-git can't reach the workstation (by design), so the workstation pulls from pve-env1.
- Key `C:\Users\[user]\.ssh\devgit_backup_pull`, no passphrase (a scheduled task can't type one). Made on the workstation in PowerShell. Don't run the `ssh-keygen` line on pve-env1: `$env:USERPROFILE` means nothing in bash, and on Sep 29 it created a stray key named `:USERPROFILE.sshdevgit_backup_pull` in admin's home on pve-env1 (deleted).
- On pve-env1, the key is in `admin`'s `~/.ssh/authorized_keys` as `command="/usr/local/bin/devgit-backup-send",restrict ssh-ed25519 ... devgit-backup-pull`. `/usr/local/bin/devgit-backup-send` only answers `list` (dev-git backup names and sizes) and `get <file>` (one dev-git backup, name checked against `^vzdump-lxc-3300-[0-9_-]+\.tar\.zst$`). Anything else prints `denied`. Verified: `whoami` and `get /etc/shadow` both denied.
- Script `C:\Users\[user]\dcollins-homelab\Scripts\powershell\pull-devgit-backup.ps1`: fetches the newest backup to `D:\Backups\dev-git` over Tailscale (pve-env1 at 100.x.x.x) using `cmd /c` redirection so the bytes aren't re-encoded, checks the size matches what pve-env1 reported, keeps the newest 3. Log at `%LOCALAPPDATA%\devgit-backup-pull.log` (on C:, so a locked D: still gets logged).
- Scheduled task `Pull dev-git backup`: weekly Sunday 1pm plus at logon, start when available. Verified both run by hand and via `Start-ScheduledTask`.
- Needs Tailscale running and D: unlocked on the workstation. First pull took about a minute over Tailscale's Dallas relay.

The backup files contain the whole container, including `app.ini` with Gitea's secret keys and the database. Keep them only on encrypted storage.

## Open items (not part of this build)
- **Backups:** done, see step 12.
- **Wazuh agent:** decided Sep 29 to add it. Needs OPNsense alias `dev_git` (10.0.50.x) and a pass rule on the DMZ interface, TCP, `dev_git` to `Wazuh_Manager` on 1514 to 1515, logging on, above the DMZ block rule. Then [[wazuh-enroll-agent]]. Not done yet.
- **Tailscale relay:** the workstation reaches dev-git through Tailscale's relay (`dfw`), not directly. Possibly OPNsense outbound NAT randomizing ports. Not investigated.
- **Uptime Kuma** on [[services-host]] can't monitor this box, because VLAN20 to DMZ is blocked.
- **Keeping a copy in the internal Gitea:** not planned. It would need a path between VLAN20 and the DMZ, which doesn't exist.

## Related
- [[dev-git]]
- [[gitea-collab-access]]
- [[tailscale-access-model]]
- [[tailscale-tailnet-policy|Apply the Tailnet Policy]]
- [[proxmox-create-lxc]]
- [[VLAN50]]
- [[pve-env1]]

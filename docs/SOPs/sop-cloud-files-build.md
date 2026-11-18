---
name: sop-cloud-files-build
type: sop
status: in-progress
---

# SOP: Build nimbus and Share It Over Tailscale

**Objective:** Stand up [[nimbus]] (Nextcloud) in the DMZ and give the collaborator access through Tailscale machine sharing. Reasoning is in [[cloud-files-design]].

**Progress (Sep 30, 2026):** steps 1 to 9 done and running. Step 10 (verify) is partly done: the collaborator loaded the HTTPS page and logged in with two-factor. The client sync test, a restore test and the isolation checks are not recorded. See [[nimbus]] for current state and open items.

**Written:** Sep 29, 2026, before the build, and updated to match what was actually run on Sep 30, 2026. Steps 1 to 3 copy the pattern from [[sop-gitea-collab-build]]. Where the plan and the build differed, the step below says what was done.

**Placeholders:** `<tailnet>` is the tailnet name (`[tailnet].ts.net` per [[dev-git]]). `<cloud-files-ts-ip>` is the 100.x address Tailscale gives the container.

## Decisions made before starting (resolved Sep 30, 2026)
1. **HTTPS: yes.** Tailscale's certificate feature requires "Enable HTTPS" on the DNS page of the admin console, and the console makes you acknowledge that machine names and the tailnet DNS name are published on a public certificate log. Daniel accepted that. See step 4.
2. **Nextcloud 34**, the previous major, not 35.0.0. Debian 13 ships PHP 8.4, which is on the supported PHP list. See [[cloud-files-design]].

## 1. Create the container (Proxmox, pve-env1)
Confirm 10.0.50.x is unused first: `ping 10.0.50.x` from [[npm-dmz]].

| Setting | Value |
|---|---|
| CT ID | 3400 |
| Hostname | nimbus |
| Unprivileged | Yes |
| Template | Debian 13 |
| Disk | 8GB on `local-lvm` |
| Cores | 2 |
| Memory / Swap | 2048 MB / 512 MB |
| Network | Same bridge as dev-git, VLAN tag 50, static `10.0.50.x/24`, gateway `10.0.50.x` |
| DNS | `10.0.50.x` (not Pi-hole, see [[sop-cloudflare-tunnel-implementation]]) |

Check pve-env1's memory on the Proxmox node summary first. It was 56% of 31GB on Sep 28, 2026.

Before starting, add the tunnel device Tailscale needs and the data disk, from the pve-env1 shell:
```bash
pct set 3400 --dev0 /dev/net/tun
pct set 3400 -mp0 local-lvm:250,mp=/srv/nextcloud-data,backup=0
pct start 3400
```
`backup=0` keeps the 250GB disk out of the Proxmox backup job. Confirm the mount inside the container with `df -h /srv/nextcloud-data`. Because the container is unprivileged, the data folder needs to be owned by the web server user (`www-data`) from inside the container. Do `chown` on the mount root non-recursively, or you get `cannot read directory lost+found: Permission denied` (harmless, the folder is owned by the host's root). Use a subfolder, `/srv/nextcloud-data/data`, as the Nextcloud data directory.

## 2. Base packages (inside the container)
```bash
apt update && apt full-upgrade -y
apt install -y curl gnupg unattended-upgrades
```

## 3. Join the tailnet
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4
```
Approve the machine under Daniel's account and write down the 100.x address. Disable key expiry on this node in the Machines page, or the collaborator loses access silently when the key expires (see [[dev-git]]).

## 4. HTTPS through Tailscale
Order matters. Do these in this order and check each result.

1. Make sure Apache answers on port 80 locally, and that the container firewall (step 7) accepts Tailscale traffic on 443.
2. Start the HTTPS proxy, which keeps running across reboots:
```bash
tailscale serve --bg --https=443 http://127.0.0.1:80
tailscale serve status
```
3. Load `https://nimbus.[tailnet].ts.net` from the workstation and confirm the login page appears.
4. Only then tell Nextcloud it is behind HTTPS:
```bash
cd /var/www/nextcloud
runuser -u www-data -- php occ config:system:set overwriteprotocol --value=https
runuser -u www-data -- php occ config:system:set overwrite.cli.url --value=https://nimbus.[tailnet].ts.net
runuser -u www-data -- php occ config:system:set trusted_proxies 0 --value=127.0.0.1
```
5. HSTS: add `Header always set Strict-Transport-Security "max-age=15552000"` to `/etc/apache2/sites-available/nextcloud.conf` right after the DocumentRoot line, then `apache2ctl configtest && systemctl reload apache2`.

What went wrong the first time: `overwriteprotocol=https` was set before `tailscale serve` was running, so the site stopped loading. Fix: `occ config:system:delete overwriteprotocol`, start the serve proxy, check the page, then set it again. the collaborator, who is on a machine share, could load the HTTPS page, so the share works with `tailscale serve`.

## 5. Install Nextcloud (inside the container)
Nextcloud's Ubuntu example uses these packages, which are the starting list here: `apache2`, `libapache2-mod-php`, a database server, and `php-gd php-mysql php-curl php-mbstring php-intl php-gmp php-bcmath php-xml php-imagick php-zip`. For PostgreSQL swap `php-mysql` for `php-pgsql`. Confirm against the current admin manual.

Download the tarball from Nextcloud's download page and verify it before extracting. The docs give both checks:
```bash
sha256sum -c nextcloud-x.y.z.tar.bz2.sha256 < nextcloud-x.y.z.tar.bz2
gpg --import nextcloud.asc
gpg --verify nextcloud-x.y.z.tar.bz2.asc nextcloud-x.y.z.tar.bz2
```
Look for `Good signature`. Then extract to `/var/www` and `chown -R www-data:www-data /var/www/nextcloud`.

**To confirm against the current docs at build time:** the Apache virtual host (needs `mod_rewrite`, with `mod_headers`, `mod_env`, `mod_dir` and `mod_mime` recommended), the exact `occ maintenance:install` options including the data directory `/srv/nextcloud-data`, the PHP memory and upload limits, `trusted_domains`, the reverse proxy and overwrite settings if HTTPS terminates at Tailscale, and cron for background jobs.

## 6. Tailnet policy (Tailscale admin console)
Done Sep 30, 2026. A host alias for nimbus, a grant for the workstation and a grant for `autogroup:shared`, each on `["tcp:80", "tcp:443"]`, plus tests: the workstation accepts nimbus:80 and :443 and denies nimbus:22, and nimbus cannot reach pve-env1, services-host, heimdall, the workstation or dev-git. Because the container firewall only accepts 443, the `tcp:80` entries are redundant. Removing them is a later cleanup. Use the dev-git grants in [[sop-gitea-collab-build]] step 4 as the template.

## 7. Lock down the container
Same as [[sop-gitea-collab-build]] step 11: container SSH disabled and masked (manage with `pct enter 3400`), nftables table `nimbus` with policy drop and no `flush ruleset` line so Tailscale's own tables survive. The rule for the served port reads `iifname "tailscale0" tcp dport 443 accept` (it was 80 until HTTPS was switched on). Check with `nft -c -f /etc/nftables.conf` before loading.

## 8. Accounts and sharing
Done Sep 30, 2026.
1. Admin account `[nimbus-admin]` created during install.
2. Regular users `collab-user` and `[admin-user]` (Daniel's own account) created. No quota. Passwords are not written here.
3. Two-factor: the `twofactor_totp` app was already enabled. Enforce it under Administration settings, Security. Confirmed working for the collaborator and the admin.
4. Machine page, Share, invite sent to the collaborator, accepted from his tailnet.
5. Done after `[admin-user]` logged in with two-factor: `occ user:disable [nimbus-admin]` (confirmed `enabled: false`). Re-enable with `occ user:enable [nimbus-admin]` when admin work is needed, then disable it again.

Admin overview warnings fixed along the way: APCu as `memcache.local` (plus `apc.enable_cli=1` for the CLI), PHP `opcache.interned_strings_buffer=16` and `apc.shm_size=128M` in `99-nextcloud.ini` drop-ins, background jobs on cron (`occ background:cron` and a `*/5` cron line for `www-data`), `maintenance_window_start` 8, `default_phone_region` US, and `memory_limit` 512M.

## 9. Backups
Design, reasons and risks are in [[nimbus-backup-design]]. What was run, in order:

**A. Proxmox job (Datacenter, Backup).** One weekly job selecting CT 3300 and CT 3400: storage `local`, `sun 03:00` (the node runs UTC), snapshot mode, ZSTD, keep last 3. The data disk is excluded by `backup=0`. Run a manual backup of CT 3400 once to test it.

**B. Data snapshot on pve-env1.** Install `/usr/local/bin/nimbus-data-snapshot` (root, mode 755; a copy is in `Scripts\bash\nimbus-data-snapshot.sh`). It turns on Nextcloud maintenance mode, runs `sync`, makes the LV snapshot `pve/snap-nimbus-data` of `pve/vm-3400-disk-2`, turns maintenance mode off, activates the snapshot with `lvchange -ay -K` and mounts it read-only at `/mnt/nimbus-data-snap`. Schedule it:
```
# /etc/cron.d/nimbus-data-snapshot
30 3 * * 0 root /usr/local/bin/nimbus-data-snapshot >> /var/log/nimbus-data-snapshot.log 2>&1
```
Run it by hand once and check that the account folders appear under `/mnt/nimbus-data-snap/data`.

**C. Restricted pull key on pve-env1.** Create the key on the workstation first (step D), then add one line to `~admin/.ssh/authorized_keys` (back the file up first):
```
command="sudo /usr/bin/rrsync -ro /mnt/nimbus-data-snap/data",restrict ssh-ed25519 <public key> nimbus-data-pull
```
and `/etc/sudoers.d/nimbus-pull` (mode 440, check with `visudo -c -f`):
```
Defaults!/usr/bin/rrsync env_keep += "SSH_ORIGINAL_COMMAND"
admin ALL=(root) NOPASSWD: /usr/bin/rrsync -ro /mnt/nimbus-data-snap/data
```
`rrsync` comes with the rsync package and needs `rsync` installed on pve-env1.

**D. Workstation pull of the files (WSL2 Ubuntu).** The D: BitLocker drive is visible in WSL at `/mnt/d` once it is unlocked. Generate a key without a passphrase in WSL (`~/.ssh/nimbus_data_pull`, comment `nimbus-data-pull`), make one manual SSH connection to pve-env1's Tailscale address to accept its host key after comparing the fingerprint, then install `~/bin/pull-nimbus-data.sh` (copy in `Scripts\bash\pull-nimbus-data.sh`). It needs `DRYRUN=1` to test without copying. The Windows scheduled task `Pull nimbus data` runs `wsl.exe -d Ubuntu -- /home/[wsl-user]/bin/pull-nimbus-data.sh`, Sundays 1:30pm plus at logon, start when available, 8 hour limit, no overlapping runs. Log: `%LOCALAPPDATA%\nimbus-data-pull.log`. First run copied about 203MB; a repeat sent 98 bytes.

**E. Workstation pull of the container backup.** On pve-env1, widen `/usr/local/bin/devgit-backup-send` so its regex and `list` loop cover `vzdump-lxc-3300-*` and `vzdump-lxc-3400-*` (keep a `.bak` copy, check with `sh -n`, and test `get` by piping to `head -c 1000 | wc -c`). On the workstation, copy `pull-devgit-backup.ps1` to `pull-nimbus-backup.ps1` with the VMID, `D:\Backups\nimbus` and log name swapped, run it once by hand (about 691 MiB; the `.partial` file looks empty in a directory listing until it finishes), then clone the dev-git scheduled task into `Pull nimbus backup` (Sundays 1pm plus logon). Re-run the dev-git task afterward to prove it still works.

## 10. Verify
- Done: the collaborator loads the HTTPS page, logs in and uses two-factor.
- Not recorded yet: the collaborator syncs a file with a Nextcloud client; the collaborator cannot reach any other port on the container; the container cannot reach the tailnet or the LAN (`nc -zv -w3 100.x.x.x 22` and `nc -zv -w3 10.0.20.x 22` from inside should both fail); data lands on the data disk, not the system disk (`df -h` before and after a large upload).
- Not tested: restoring either backup.

## Open items
- Wazuh agent (OPNsense rule plus enrollment), same as [[dev-git]].
- Nightly database dump inside the container, since snapshot backups of a running database are crash-consistent only.
- Uptime Kuma cannot monitor this box, because VLAN20 to DMZ is blocked.
- Reboot test, password changes and the rest of the list in [[nimbus]].
- Backup risks and follow-ups are in [[nimbus-backup-design]].

## Related
- [[nimbus]]
- [[cloud-files-design]]
- [[sop-gitea-collab-build]]
- [[nimbus-backup-design]]
- [[dev-git-backup-strategy]]
- [[VLAN50]]
- [[pve-env1]]

## Build notes (Sep 30, 2026)
- Wizard requires a root password or SSH key. A dedicated `nimbus` key was used, and container SSH is disabled and masked afterward.
- Running `pct set 3400 -mp0 local-lvm:250,...` twice leaves an `unused0` volume. Remove with `pct set 3400 --delete unused0`.
- `occ maintenance:install --help` gave the option list. Command used: `--database pgsql --database-name nextcloud --database-user nextcloud --database-pass "$(cat /root/nextcloud-db-pass)" --admin-user [nimbus-admin] --admin-pass "$ADMINPASS" --data-dir /srv/nextcloud-data/data`, run with `runuser -u www-data -- php occ ...`.
- Apache: site `nextcloud.conf` (DocumentRoot `/var/www/nextcloud/`, `AllowOverride All`, `Dav off`), default site disabled, modules `rewrite headers env dir mime` enabled, `systemctl restart apache2`.
- Trusted domains and `overwrite.cli.url` set with `occ config:system:set`.
- While only port 80 was open, browsers that upgrade typed addresses to HTTPS timed out. That went away once HTTPS was served in step 4.

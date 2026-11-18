---
name: nimbus
type: lxc
vmid: 3400
node: pve-env1
ip: 10.0.50.x
vlan: VLAN50
status: live
role: Nextcloud file storage shared between Daniel and the collaborator, reached over Tailscale only
---

Built Sep 30, 2026. Reasoning is in [[cloud-files-design]], build steps are in [[sop-cloud-files-build]], backups are in [[nimbus-backup-design]].

Debian 13 unprivileged LXC on [[pve-env1]] in the DMZ ([[VLAN50]]). Nextcloud 34 installed natively (no Docker) with Apache, PHP 8.4 and PostgreSQL 17. Reached at `https://nimbus.[tailnet].ts.net` from the tailnet only.

## Current state (Sep 30, 2026)
- **Container:** CT 3400, 2 cores, 2GB RAM, 512MB swap, nesting off, `/dev/net/tun` passed through, static 10.0.50.x/24 on VLAN50 (tag 50), DNS 10.0.50.x. The network firewall checkbox (`firewall=1`) was left on from the wizard and has not been compared with [[dev-git]].
- **Disks:** rootfs 8GB (`vm-3400-disk-0`) and a 250GB data disk (`vm-3400-disk-2`) mounted at `/srv/nextcloud-data` with `backup=0`. Nextcloud's data directory is the subfolder `/srv/nextcloud-data/data`.
- **Access:** dedicated `nimbus` root SSH key from the create wizard. Container SSH (`ssh.service`, `ssh.socket`) disabled and masked. Manage with `pct enter 3400` from [[pve-env1]].
- **Tailscale:** joined as `nimbus`, address 100.x.x.x, key expiry disabled. Shared to the collaborator with machine sharing.
- **HTTPS:** `tailscale serve --bg --https=443 http://127.0.0.1:80`. Tailscale's HTTPS certificates were enabled on the tailnet, which publishes machine and tailnet names on a public certificate log. Daniel chose this. the collaborator reaches the HTTPS page through the machine share (confirmed Sep 30).
- **Tailnet policy:** grants for the workstation and `autogroup:shared` on `tcp:80` and `tcp:443`. Tests: the workstation reaches nimbus:80 and :443 and not :22; nimbus cannot reach pve-env1, services-host, heimdall, the workstation or dev-git. The `tcp:80` grants are now redundant with the container firewall and are left for later cleanup. Oct 23, 2026: added host alias `[admin-user]-worktop` (100.x.x.x) with a grant to nimbus on `tcp:443` only, plus a test that it reaches nimbus:443 and not nimbus:22, nimbus:80, pve-env1:22, pve-env1:8006 or workstation:22. Before this, the laptop was on the tailnet but could not connect, because the default-deny policy only allowed the workstation and `autogroup:shared` as sources. Login from [admin-user]-worktop confirmed working the same day.
- **Container firewall:** nftables table `nimbus`, policy drop. Accepts only Tailscale traffic to port 443, plus ICMP and Tailscale's UDP 41641. Port 80 was closed Sep 30 after `tailscale serve` started proxying to `127.0.0.1:80`.
- **Nextcloud settings:** `overwriteprotocol` https, `overwrite.cli.url` `https://nimbus.[tailnet].ts.net`, `trusted_proxies` 127.0.0.1, trusted domains `localhost`, `nimbus.[tailnet].ts.net`, `100.x.x.x`. APCu as local cache, background jobs on cron (`*/5` as `www-data`), maintenance window at 8 UTC, default phone region US. PHP drop-ins `99-nextcloud.ini` for Apache (`opcache.interned_strings_buffer=16`, `apc.shm_size=128M`) and CLI (`apc.enable_cli=1`). `memory_limit` 512M.
- **HSTS:** `Header always set Strict-Transport-Security "max-age=15552000"` added to `/etc/apache2/sites-available/nextcloud.conf` after the DocumentRoot line. Whether the admin overview warning cleared was not confirmed.
- **Accounts:** `[nimbus-admin]` (admin, disabled Sep 30, 2026, see below), `collab-user` (regular user, logged in, two-factor set up), `[admin-user]` (Daniel's regular account, logged in Sep 30 with TOTP two-factor; no backup codes generated). No quotas. Passwords are not recorded in the vault. Two-factor (TOTP) is enforced for all users in Administration settings, Security, and confirmed working for the collaborator and the admin.
- **Secrets:** DB password and the original admin password are in root-only files `/root/nextcloud-db-pass` and `/root/nextcloud-admin-pass`. To be changed and stored in Vaultwarden.
- **Version:** Nextcloud 34 (`latest-34.tar.bz2`, SHA256 matched, GPG signature good from Nextcloud Security, fingerprint 2880 6A87 8AE4 23A2 8372 792E D758 99B9 A724 937A).
- **Backups:** weekly Proxmox backup of the container plus a weekly snapshot and pull of the data files, both to the workstation's D: drive. First pulls done and scheduled tasks tested. See [[nimbus-backup-design]].

## Admin account lockdown (done Sep 30, 2026)
`[nimbus-admin]` is disabled (`occ user:disable [nimbus-admin]`, confirmed `enabled: false`). `[admin-user]` was checked first: enabled, logged in, TOTP two-factor on, and Daniel confirmed he can still log in. `[admin-user]` is a regular user, so there is no web admin while the account is disabled. To change admin settings, run `occ user:enable [nimbus-admin]` from pve-env1 (`sudo pct exec 3400 -- runuser -u www-data -- php /var/www/nextcloud/occ user:enable [nimbus-admin]`), do the work, then disable it again. If Daniel loses his authenticator, `occ twofactorauth:disable [admin-user]` restores access.

## Not done yet
- Nextcloud desktop and phone client test over HTTPS.
- Change the admin and DB passwords and store them in Vaultwarden.
- Wazuh agent (OPNsense rule plus enrollment), same as [[dev-git]].
- Reboot test: nftables, `tailscale serve`, and the backup snapshot mount after a restart.
- Nightly database dump inside the container.
- Identify the listener on `127.0.0.1:25` (Postfix was seen by `ss -ltn`; not needed for this role).
- Uptime Kuma cannot monitor this box, because VLAN20 to DMZ is blocked.
- Confirm the HSTS overview warning cleared, and compare `firewall=1` with dev-git (`sudo pct config 3300`).
- Thin pool `pve/data` is over-provisioned with no autoextend protection. See [[nimbus-backup-design]].

## Considered, not done: single sign-on through Authentik
Daniel asked whether the OAuth 2.0 clients page in [[authentik]] is where SSO would connect, and pushed back that the value is for an organization with users around the world, not for two people. Nextcloud would use the `user_oidc` app. Two constraints in this build: the DMZ cannot reach RFC1918 addresses (Authentik is at 10.0.30.x on VLAN30), and the collaborator's browser cannot reach Authentik. Not built. The open question is whether an organization deployment would be a separate build or a change to nimbus.

## Related
- [[pve-env1]]
- [[VLAN50]]
- [[cloud-files-design]]
- [[nimbus-backup-design]]
- [[dev-git]]
- [[Tailscale]]

Hostname is `nimbus`, chosen deliberately by Daniel on Sep 30, 2026. Earlier drafts of these notes called it `cloud-files`; the design and SOP notes keep that name for the project, the host is `nimbus`.

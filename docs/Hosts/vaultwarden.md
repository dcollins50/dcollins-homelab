---
name: vaultwarden
type: container
vmid: n/a
node: pve-env1
ip: 10.0.20.x
vlan: VLAN20
status: live
role: Self-hosted password manager
---

Runs as a Docker container on [[services-host]] (VM 200) and shares its IP. Published as `8080:80`. On the Docker networks `reverse-proxy-net` (so the internal NPM can reach it) and `security-net`. TOTP two step login is on for the main account, see [[vaultwarden-enable-mfa]].

Holds every homelab login, organized under `homelab/` by function, see [[vaultwarden-add-homelab-login]] and [[vaultwarden-integration]].

## Not yet recorded
Checked against the live container, then filled in here. Use [[vaultwarden-find-version-and-deployment]].
- Real Vaultwarden version: unknown. The server reports `2025.6.0`, which is the Bitwarden compatible string, not the Vaultwarden release.
- Image and tag: unknown.
- Started from compose or `docker run`: unknown.
- Data folder on the host: unknown.
- Setting names in use (never values): unknown.
- Offline backup: never configured.

## Runbooks
- [[vaultwarden-find-version-and-deployment]]
- [[vaultwarden-backup-data]]
- [[vaultwarden-restore-backup]]
- [[vaultwarden-schedule-offline-backup]]
- [[vaultwarden-update-container]]
- [[vaultwarden-enable-admin-page]]
- [[vaultwarden-enable-mfa]]
- [[vaultwarden-disable-signups-and-invite-users]]
- [[vaultwarden-configure-smtp]]
- [[vaultwarden-npm-proxy-host]]
- [[vaultwarden-enable-file-logging]]
- [[vaultwarden-configure-fail2ban]]
- [[vaultwarden-harden-container]]
- [[vaultwarden-configure-sso-authentik]]
- [[vaultwarden-monitor-uptime-kuma]]
- [[vaultwarden-bitwarden-cli-setup]]
- [[vaultwarden-use-cli-in-scripts]]
- [[vaultwarden-ssh-agent-windows]]
- [[vaultwarden-ssh-agent-linux]]
- [[vaultwarden-store-ssh-key-passphrases]]
- [[vaultwarden-add-homelab-login]]
- [[vaultwarden-export-vault]]
- [[vaultwarden-import-from-bitwarden-cloud]]
- [[vaultwarden-emergency-access]]
- [[vaultwarden-recover-access]]

## Related
- [[services-host]]
- [[VLAN20]]

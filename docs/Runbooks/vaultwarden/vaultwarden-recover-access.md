---
name: vaultwarden-recover-access
type: runbook
tool: vaultwarden
---

What to do if you cannot get into Vaultwarden. Written as a decision list. It only claims what the existing notes and official docs support. Anything else is flagged.

## If you lost your authenticator (two step code)
- Use the recovery code saved when TOTP was enabled. [[vaultwarden-enable-mfa]] says to keep it outside Vaultwarden, because without it the only way back is admin level database access.
- Where the recovery code is stored: <FILL IN>. Confirm it is findable today.

## If the server is down or broken
- Restore from backup: [[vaultwarden-restore-backup]]. If no backup exists, see [[vaultwarden-schedule-offline-backup]] and fix that first.
- Clients keep an encrypted local cache of the vault. A signed in client can often still show items while the server is offline. Do not log that client out until the server is back. Confirm on your own client before relying on this.

## If you forgot the master password
- The vault is encrypted with it. Vaultwarden cannot reset it and still keep the data. Check emergency access: [[vaultwarden-emergency-access]].
- A password hint only exists if hints are enabled, and [[vaultwarden-harden-container]] recommends turning them off.

## If SSO breaks sign in
- If `SSO_ONLY` was turned on, email and password sign in is disabled. Unset it in the container environment and re-create the container. See [[vaultwarden-configure-sso-authentik]].

## If the admin token is lost
- Set a new hashed token, see [[vaultwarden-enable-admin-page]] and restart.

## Prepare now
- Keep the TOTP recovery code, the admin password, and the master password hint somewhere physical and separate.
- Keep one tested offline backup.

## Related
- [[vaultwarden]]

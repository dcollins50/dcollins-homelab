---
name: vaultwarden-backup-data
type: runbook
tool: vaultwarden
---

Take a one off backup of Vaultwarden's data folder. Never configured as a recurring job yet, see [[vaultwarden-schedule-offline-backup]] for that.

## What to back up (official guidance)

- Required: `db.sqlite3` and the `attachments/` folder.
- Recommended: `config.json` (settings saved from the admin page) and the `rsa_key*` files (token signing keys).
- Optional: `sends/` and `icon_cache/`.

Simplest safe habit: back up the whole data folder, and copy the database with the SQLite online backup so it is consistent.

## Steps

1. Find the data folder on the host. See [[vaultwarden-find-version-and-deployment]]. Below it is written as `<DATA>`.

2. Take a consistent copy of the database. Vaultwarden's own docs say to use SQLite's backup method and not a plain file copy of a running database. Version 1.32.1 and newer have a built in command:
   ```bash
   docker exec -it <container> /vaultwarden backup
   ```
   Check the output on the first run to see the filename and where it was written. If your version is older than 1.32.1, update first or use `sqlite3`:
   ```bash
   sqlite3 <DATA>/db.sqlite3 ".backup '/path/to/backups/db-$(date '+%Y%m%d-%H%M').sqlite3'"
   ```

3. Archive the folder and the database copy together:
   ```bash
   sudo tar -czf /tmp/vw-backup-$(date +%Y%m%d-%H%M).tar.gz -C <DATA> .
   ```

4. Copy the archive off [[services-host]] to a separate machine, then confirm it arrived and is not empty:
   ```bash
   ls -lh /tmp/vw-backup-*.tar.gz
   ```
   A backup that lives only on the same disk does not count. Target host: <FILL IN: where backups go>.

5. Treat the archive as a secret. Vault items are encrypted, but the archive also holds `config.json` and the signing keys, so it should not sit unprotected.

6. Delete the temporary copy from `/tmp` once it is safely off the host.

## Related
- [[vaultwarden-restore-backup]]
- [[vaultwarden-schedule-offline-backup]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault

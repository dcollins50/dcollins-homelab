---
name: vaultwarden-restore-backup
type: runbook
tool: vaultwarden
---

Restore Vaultwarden's data from a backup taken with [[vaultwarden-backup-data]]. Do this on [[services-host]] or on a rebuilt replacement.

## Steps

1. Stop Vaultwarden so nothing is writing to the data folder:
   ```bash
   docker stop <container>
   ```

2. Move the current data folder aside so it can be put back if the restore goes wrong:
   ```bash
   sudo mv <DATA> <DATA>.old-$(date +%Y%m%d)
   sudo mkdir <DATA>
   ```

3. Unpack the backup into the data folder:
   ```bash
   sudo tar -xzf /path/to/vw-backup-<stamp>.tar.gz -C <DATA>
   ```
   Official guidance: replace each file or folder in the data folder with its backed up version.

4. If the database came from a `.backup` or `VACUUM INTO` copy, make sure the live name is `db.sqlite3` and delete any leftover `db.sqlite3-wal` file first. The official docs warn that a stale WAL file can corrupt the database:
   ```bash
   sudo rm -f <DATA>/db.sqlite3-wal
   ```

5. Match ownership to what the container expects. If the container runs as a non-root user, the files must be owned by that UID. <FILL IN: UID the container runs as, or root if default>.

6. Start it and watch the log:
   ```bash
   docker start <container>
   docker logs -f <container>
   ```

7. Verify. Log in on the web vault, check that item counts and a few attachments look right, and check `server.version` at `/api/config`.

8. Keep the `.old-` folder until you are sure everything is fine, then remove it.

## Related
- [[vaultwarden-backup-data]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault

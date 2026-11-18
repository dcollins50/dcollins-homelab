---
name: vaultwarden-schedule-offline-backup
type: runbook
tool: vaultwarden
---

Run the Vaultwarden backup on a schedule and ship it off [[services-host]]. This is the open item flagged since Sep 13, 2026: an offline backup was never configured.

Everything here uses the same file list and backup method as [[vaultwarden-backup-data]]. Fill in the destination before using it.

## Decide first
- Destination host or storage: <FILL IN>
- Transfer method (rsync over SSH, scp, or a mounted share): <FILL IN>
- How many copies to keep: <FILL IN>
- Where the encryption passphrase for the archive is stored (not inside Vaultwarden alone): <FILL IN>

## Steps

1. Write the script on [[services-host]]. Heredoc so indentation survives:
   ```bash
   sudo tee /usr/local/bin/vw-backup.sh > /dev/null <<'SCRIPT'
   #!/usr/bin/env bash
   set -euo pipefail
   DATA="<FILL IN: data folder>"
   OUT="/var/backups/vaultwarden"
   STAMP="$(date +%Y%m%d-%H%M)"
   mkdir -p "$OUT"
   docker exec <container> /vaultwarden backup
   tar -czf "$OUT/vw-$STAMP.tar.gz" -C "$DATA" .
   find "$OUT" -name 'vw-*.tar.gz' -mtime +14 -delete
   SCRIPT
   sudo chmod 700 /usr/local/bin/vw-backup.sh
   ```
   Check on the first manual run where `/vaultwarden backup` writes its file, and adjust.

2. Add the off-host copy as a last line of the script. Example with rsync over SSH, using a key that is only allowed to write backups:
   ```bash
   rsync -a "$OUT/vw-$STAMP.tar.gz" <FILL IN: user@host:/path/>
   ```

3. Schedule it. Example, daily at 03:10, in root's crontab (`sudo crontab -e`):
   ```
   10 3 * * * /usr/local/bin/vw-backup.sh >> /var/log/vw-backup.log 2>&1
   ```

4. Run it once by hand and confirm the archive exists locally and on the destination.

5. Wire a failure alert. Uptime Kuma can watch a push monitor that the script calls on success, so a missed run alerts through [[ntfy]]. See [[uptime-kuma-wire-notification]].

6. Test a restore from the off-host copy at least once, using [[vaultwarden-restore-backup]]. An untested backup is not a backup.

## Related
- [[vaultwarden-backup-data]]
- [[vaultwarden-restore-backup]]
- [[vaultwarden]]

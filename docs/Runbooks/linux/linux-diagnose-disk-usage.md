---
name: linux-diagnose-disk-usage
type: runbook
tool: linux
---

Find what's actually consuming disk space on a full or near-full filesystem, by drilling down from the top rather than guessing. General pattern, not tied to any one tool, used independently to diagnose two separate incidents (wazuh-manager filling from an unrotated alert log, and a later Sep 14 alert-flood spike on the same host).

## Procedure

1. **Confirm which filesystem is actually full**, don't assume it's `/`:
   ```bash
   df -h
   ```

2. **Drill down one level at a time from the full mount point**, sorted largest-first:
   ```bash
   du -h --max-depth=1 / 2>/dev/null | sort -rh | head -20
   ```
   Repeat on whichever directory shows the largest number, going one level deeper each time (`/var`, then `/var/ossec`, then `/var/ossec/logs`, etc.), until the actual offending file or directory is found. Don't jump straight to a deep guess, each level narrows it down and confirms the trail is actually leading somewhere real rather than assuming based on general knowledge of what's usually large.

3. **Once narrowed to a directory, find specific large files** rather than continuing to drill by directory:
   ```bash
   find <directory> -type f -size +500M -exec ls -lh {} \;
   ```
   Adjust the size threshold to whatever makes sense for the scale of the problem, `+500M` worked for a multi-gigabyte log file, a smaller environment might need `+100M` or similar to surface anything meaningful.

4. **Check file timestamps against known events** (a cluster shutdown, a restart, a config change) before assuming steady organic growth. In both cases this pattern was used for, the large files were the product of a short burst tied to a specific event (a mid-write file that never rotated when the cluster went down; a 30-minute alert storm right after a manager restart), not gradual accumulation. Confirming this changes what "safe to delete" and "will this recur" actually mean.

## Notes

- `du -h --max-depth=1` is deliberately shallow on purpose, each call is fast and each result tells you exactly where to look next, rather than running a single slow full-depth scan and trying to read a huge unsorted output.
- Deleting/truncating files on a filesystem that's already at 100% can still work even with zero free space (e.g. `truncate -s 0 <file>` deallocates blocks in place rather than needing space to write a new file), don't assume a full disk blocks its own cleanup.

## Related
- [[wazuh-manager]]
- [[incident-2026-09-13-elastic-log-shipping-outage]]

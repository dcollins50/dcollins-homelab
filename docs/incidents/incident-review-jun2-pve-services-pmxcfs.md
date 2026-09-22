# Incident: pve-services Lost Quorum (pmxcfs Database I/O Error, Corosync Exit 21)

**Date:** 2026-05-30 (first fault) / 2026-06-02 (corosync exit) / ~2026-06-05 (found)
**Systems affected:** pve-services (10.0.0.11), pmxcfs (pve-cluster), corosync
**Status:** Diagnosed. Resolution not documented.

## Summary

The Proxmox UI showed pve-services as having lost quorum while the node itself was still up and reachable over SSH. It had been running continuously since March 21. Corosync had exited on June 2 with status 21 (`CS_ERR_LIBRARY`). The underlying fault was three days earlier: on May 30, pmxcfs hit a disk I/O error while committing to its SQLite database, stopped its cluster connection, and left a 4 MB write-ahead log that was never merged back. The rest of the cluster ran without pve-services's vote for more than two days before anyone noticed.

## Timeline

- **Mar 21 14:16** pve-services boots. Stays up from here on.
- **May 28 to 29** Several short corosync link drops to hosts 1, 3 and 4, all recovering within seconds. May 29 21:35 all four members rejoin.
- **May 30 05:56** Last write to `config.db`.
- **May 30 06:01:06** pmxcfs: `commit transaction failed: disk I/O error`, then `rollback transaction failed`, then `serious internal error - stop cluster connection`. At the same second corosync logs `CPG can't mcast to group pve_dcdb_v1 ... error:12`.
- **May 30 06:01:07** pmxcfs: `can't initialize service`. `config.db-wal` last written at 06:01, 4,148,872 bytes.
- **Jun 2 11:22:34 CDT** Corosync exits. `Result=exit-code`, `ExecMainStatus=21`.
- **~Jun 5** Lost quorum noticed in the Proxmox UI. Investigation starts.

## Investigation

| Check | Result |
|-|-|
| Uptime / `who -b` | 75 days, booted 2026-03-21 14:16. The node never went down. |
| Memory, disk, CPU | All normal. Root filesystem 10% used. |
| `journalctl` for the June 2 window | Returns nothing or aborts with `SIGBUS handling failed: Value too large for defined data type`. The journal covering the crash is unreadable. |
| `journalctl --list-boots` | `No boot found.` |
| `systemctl status pve-cluster` | Running since Mar 21, with the May 30 06:01 critical errors as the last entries. |
| `/var/lib/pve-cluster/` | `config.db` 98 KB (May 30 05:56), `config.db-wal` 4 MB (May 30 06:01). |
| `pvecm status` | Config version 7, then `Cannot initialize CMAP service`. |

## Root Cause

Proximate cause: pmxcfs's SQLite database on pve-services was left in a broken state by a disk I/O error on May 30. With the cluster filesystem out of the cluster, corosync eventually failed with `CS_ERR_LIBRARY` on June 2.

Unknown: what caused the disk I/O error, and what triggered the June 2 corosync exit. The journal for that window is corrupted.

## Recovery

The chat that diagnosed this ended while checking `/var/lib/pve-cluster/` on the other three nodes (via `admin` with sudo, since root SSH is disabled) to find a clean `config.db` to recover from. Chat history does not show how or whether the recovery was finished.

## Open items

- Document how pve-services was actually recovered, or confirm whether it is still affected.
- Find the source of the disk I/O error. The disk and filesystem under `/var/lib/pve-cluster` have not been checked.
- Clean up the corrupted journal files on pve-services.
- Quorum loss went unnoticed for 2+ days. There is no alert on node quorum state.

## Related

- [incident-review-may27-pve-env1-silent-shutdown.md](incident-review-may27-pve-env1-silent-shutdown.md), a separate quorum loss on a different node one week earlier.

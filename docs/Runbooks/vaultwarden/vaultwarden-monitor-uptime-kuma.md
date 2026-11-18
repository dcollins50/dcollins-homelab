---
name: vaultwarden-monitor-uptime-kuma
type: runbook
tool: vaultwarden
---

Monitor Vaultwarden with Uptime Kuma and get alerted through [[ntfy]]. Part of the monitoring item in the Oct 3, 2026 scope. Whether a Vaultwarden monitor already exists in Kuma is <FILL IN>, because the Sep 13, 2026 firewall work already treated Vaultwarden on port 8080 as a monitored service.

Kuma runs on [[services-host]], the same host as Vaultwarden. A host outage takes both down at once, so a monitor on the same host cannot tell you about that. Consider a second check from another host: <FILL IN>.

## Steps
1. Kuma, Add New Monitor, type HTTP(s).
2. URL: `https://<vaultwarden address>/api/config`. It should answer with JSON and status 200.
3. Optional: use a keyword monitor and look for the word `server` in the body.
4. Heartbeat interval and retries: <FILL IN>. Kuma alerts only after the retry count, not on the first failed check.
5. Turn on certificate expiry notification for this monitor. It is opt in per monitor, with warnings at 21, 14 and 7 days. This matters once Vaultwarden has an internal CA certificate.
6. Attach the ntfy notification, scoped to the write only `kuma-publisher` account. See [[uptime-kuma-wire-notification]].
7. Test by stopping the container briefly and confirming a down and a back up notice arrive. Do this at a quiet time.

## Also worth watching
- Backup freshness: a Kuma push monitor that the backup script calls on success, see [[vaultwarden-schedule-offline-backup]].
- Disk space on [[services-host]], since logs and backups live there.

## Related
- [[uptime-kuma-wire-notification]]
- [[vaultwarden]]

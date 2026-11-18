---
name: vaultwarden-enable-file-logging
type: runbook
tool: vaultwarden
---

Make Vaultwarden write a log file, for fail2ban and for log shipping to the SOC stack. Whether it already logs to a file on [[services-host]] is <FILL IN>.

## Settings (official wiki)
- `LOG_FILE`: path of the log file. When set, messages go to both stdout and the file.
- `EXTENDED_LOGGING`: must be `true` for `LOG_LEVEL` and syslog to take effect.
- `LOG_LEVEL`: `trace`, `debug`, `info`, `warn`, `error` or `off`. Default `info`. Needs `EXTENDED_LOGGING=true`.
- `USE_SYSLOG`: `true` sends logs to syslog as well. Needs `EXTENDED_LOGGING=true`.

## Steps
1. Add to the container environment, using a path inside the mounted data folder so the file survives container re-creation:
   ```
   LOG_FILE=/data/vaultwarden.log
   ```
2. Re-create the container.
3. On the host, the file is in the data folder as `vaultwarden.log`. Confirm lines are appearing:
   ```bash
   sudo tail -f <DATA>/vaultwarden.log
   ```
4. Rotate it so it cannot fill the disk. A previous incident on this host was a log that grew unrotated (see [[incident-2026-09-13-elastic-log-shipping-outage]]). Example `/etc/logrotate.d/vaultwarden`:
   ```bash
   sudo tee /etc/logrotate.d/vaultwarden > /dev/null <<'LR'
   <FILL IN: data folder>/vaultwarden.log {
       weekly
       rotate 4
       compress
       missingok
       notifempty
       copytruncate
   }
   LR
   ```
5. To ship into the SOC stack, add the file as a Wazuh agent log source, per [[sop-log-source-onboarding]].
6. Redact `access_token` values in any proxy access logs, per the hardening guide ([[vaultwarden-harden-container]]).

## Viewing without a log file
`docker logs <container>`.

## Related
- [[vaultwarden-configure-fail2ban]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Logging

**HOMELAB INFRASTRUCTURE**

Incident Review & Lessons Learned

May 9-11, 2026  |  Heimdall Log Pipeline Failure

| **Date Opened** | May 9, 2026 |
| --- | --- |
| **Date Closed** | May 11, 2026 |
| **Author** | Daniel Collins |
| **Affected Systems** | Heimdall (RPi5), soc-stack (10.0.10.10), Logstash, Elasticsearch |
| **Severity** | Medium -- no data loss, no security breach; log visibility lost for ~50 hours |
| **Status** | Resolved |

# **Background**

During a routine infrastructure session on May 9, the objective was to extend the Heimdall rsyslog pipeline to forward Pi-hole query logs from `/var/log/pihole/pihole.log` into Logstash using the rsyslog `imfile` module. A new drop-in config file was created, rsyslog was restarted, and log forwarding silently broke. What followed was a two-day incident involving three compounding issues: a config conflict that broke rsyslog forwarding, an orphaned debug process left running at the end of the session that masked the broken state and eventually died silently, and a pre-existing Logstash timezone misconfiguration that had been causing all Heimdall log timestamps to be stored 5 hours behind actual event time since the pipeline was first established.

# **Session Timeline**

## **Issue 1 -- rsyslog Config Conflict Breaks Forwarding**

The Pi-hole imfile configuration added to `/etc/rsyslog.d/10-pihole.conf` included a `global(workDirectory=...)` directive. The base `/etc/rsyslog.conf` already contained a legacy `$WorkDirectory` directive for the same setting. rsyslog does not allow mixing legacy and RainerScript-style directives for the same global parameter. This conflict caused rsyslog to silently stop forwarding after the first restart at approximately 10:15 AM CDT on May 9.

The broken `10-pihole.conf`:

```
module(load="imfile" PollingInterval="10")
global(workDirectory="/var/spool/rsyslog")    # conflicts with legacy $WorkDirectory
input(type="imfile"
      File="/var/log/pihole/pihole.log"
      Tag="pihole"
      Severity="info"
      Facility="local7"
      reopenOnTruncate="on"
      PersistStateInterval="10")
```

The `global(workDirectory=...)` line is the conflicting directive. The `10-pihole.conf` file was removed during the troubleshooting session before this was isolated, which obscured the root cause and led to an extended circular troubleshooting loop through the rest of the day.

- Multiple rsyslog.conf changes were made attempting to resolve the issue, none of which addressed the actual conflict
- `sudo rsyslogd -nd` was run as a debug step and not terminated when the session ended
- The session concluded at approximately 16:51 CDT with rsyslog restarted and logs appearing to flow

## **Issue 2 -- Orphaned Debug Process Causes Silent Log Drop**

The `sudo rsyslogd -nd` instance left running at the end of the May 9 session (PID 195144) competed with the systemd-managed rsyslog instance (PID 195161) for the journald socket at `/run/systemd/journal/syslog`. The debug instance held the socket in CONNECTED state and was the active consumer of journald messages. The systemd-managed instance held the socket in UNCONNECTED state and was writing to local log files only, not forwarding to Logstash.

```
PID 195144 (debug instance -- sudo rsyslogd -nd):
  fd 4: /run/systemd/journal/syslog  DGRAM (CONNECTED)   -- consuming all messages
  stdout/stderr: /dev/pts/4 (deleted)                     -- output going nowhere

PID 195161 (systemd instance -- rsyslogd -n -iNONE):
  fd 3: /run/systemd/journal/syslog  DGRAM (UNCONNECTED)  -- not receiving from journald
  fd 8: /var/log/syslog                                    -- writing locally only
```

The debug instance successfully forwarded logs to Logstash until approximately 4:17 AM CDT on May 10, at which point it stopped. The deleted terminal session (`/dev/pts/4`) is the likely cause. After that, neither process forwarded anything and the log gap began.

The ghost process was identified on May 10 using `pgrep -a rsyslog` and `lsof -p <pid> | grep syslog`. It was killed, `syslog.socket` was cycled, and `systemd-journald` was restarted to re-initialize syslog forwarding.

- Resolution: `sudo kill 195140 195143 195144`
- Resolution: Stop and start `syslog.socket` before restarting rsyslog
- Resolution: `sudo systemctl restart systemd-journald && sudo systemctl restart rsyslog`
- Verified: Single clean rsyslogd process, socket acquired, UDP packets confirmed reaching Logstash via tcpdump

## **Issue 3 -- Logstash Timezone Misconfiguration (Pre-existing)**

After the pipeline was restored, a persistent 5-hour gap was observed between log event timestamps in Kibana and actual event time. The Logstash syslog input for Heimdall (port 5146) had no `timezone` parameter configured. rsyslog sends syslog timestamps in CDT (UTC-5) format with no timezone indicator per RFC 3164. Logstash defaulted to treating these timestamps as UTC, storing every Heimdall log event 5 hours earlier than it actually occurred.

This issue predated the May 9 incident and had been present since the Heimdall pipeline was first built. It was not detected until timestamp accuracy was verified during May 11 recovery confirmation.

- Resolution: Added `timezone => "America/Chicago"` to the Heimdall syslog input block in `/etc/logstash/conf.d/proxmox.conf`
- Resolution: `sudo systemctl restart logstash`
- Verified: New log events indexed with correct timestamps matching actual CDT event time

# **Root Cause Analysis**

## **Config Directive Conflict in 10-pihole.conf**

The `global(workDirectory=...)` directive in the new Pi-hole config conflicted with the existing legacy `$WorkDirectory` directive in rsyslog.conf. rsyslog silently stopped forwarding rather than producing an error. The conflicting file was removed during troubleshooting before it could be identified as the cause, which prevented clean root cause isolation.

## **Debug Process Not Terminated Before Session End**

`sudo rsyslogd -nd` launches a fully functional rsyslog instance. On a systemd socket-activated system, it competes for the journald syslog socket with the managed instance. The first process to hold the socket in CONNECTED state becomes the active consumer. The managed instance receives nothing and produces no error. There is no built-in protection against this.

## **Timezone Not Set on Logstash Syslog Input**

The Heimdall syslog input in Logstash was configured without a timezone parameter. All historical Heimdall log documents in Elasticsearch retain incorrect timestamps. Reindexing was not performed.

# **What I Could Have Done Better**

## **End-to-End Pipeline Verification Before Closing Sessions**

After any rsyslog config change and restart, the forwarding pipeline should be verified end-to-end before the session is closed. Confirming rsyslog is running is not the same as confirming it is forwarding. The verification should include checking the Elasticsearch document count and confirming the latest timestamp matches current time.

## **Process Cleanup Before Ending Debug Sessions**

Any manually launched rsyslogd instance must be explicitly killed before ending a session. Running `pgrep -a rsyslog` as a final check before closing any rsyslog troubleshooting session would have prevented the entire May 10 gap.

## **Timestamp Accuracy Verification at Pipeline Build Time**

When the Heimdall log pipeline was first built, the timestamp accuracy was never verified end-to-end. Comparing a known event time against its `@timestamp` value in Kibana would have caught the 5-hour offset immediately. This check should be part of any new pipeline validation.

## **Preserve Config Files During Troubleshooting**

The `10-pihole.conf` file was moved during the session before its contents were identified as the root cause. Config files under investigation should be renamed rather than removed so the state is preserved for analysis.

# **Process Improvements**

- **Run pipeline health checks after every rsyslog change before closing the session**

  - Confirm single rsyslogd process with `pgrep -a rsyslog`
  - Confirm socket acquired cleanly with `sudo systemctl status rsyslog --no-pager -l`
  - Confirm UDP packets reaching Logstash with tcpdump on soc-stack
  - Confirm document count increasing and latest timestamp matching current UTC time

- **Kill all manually launched rsyslogd instances before ending any session**

  - Run `pgrep -a rsyslog` as a final step and kill any non-systemd instances
  - Use `sudo journalctl -fu rsyslog` for live log monitoring instead of launching a debug instance

- **Verify timestamp accuracy at pipeline build time**

  - Generate a log event at a known time, then confirm `@timestamp` in Kibana matches
  - Confirm syslog input timezone matches the sending system's local timezone

- **Rename rather than remove config files under investigation**

  - Use `.bak` suffix rather than deleting or moving out of `/etc/rsyslog.d/`

- **Implement automated alerting for log ingestion drop-off**

  - A 30-minute document count check on the heimdall-logs index would have detected the May 10 gap within one check window

# **Remaining Open Items**

| **Item** | **Notes** | **Status** |
| --- | --- | --- |
| Pi-hole log ingestion via rsyslog imfile | Original May 9 goal; deferred | **Open** |
| Automated alerting for Heimdall log ingestion drop-off | No monitoring currently exists | **Open** |

github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution

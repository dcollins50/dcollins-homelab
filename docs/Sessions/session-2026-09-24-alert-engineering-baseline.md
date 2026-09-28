---
name: session-2026-09-24-alert-engineering-baseline
type: session
date: 2026-09-24
---

Start of alert engineering work on the Wazuh stack ([[SOC-Stack]]). Goal: cut day-to-day triage and catch silent breakage. Raw log, not tidy.

## Baseline (7 days to Sep 24)
- 4,657 alerts. Top 20 rules cover 4,649.
- Vulnerability detector rules 23503 to 23506: 3,354 alerts (72%), all on Sep 24 UTC (one-day burst), split evenly between services-host and soc-stack, kernel 6.8.0-142. Cause not identified yet.
- Without that burst the week is 1,303 alerts. Rule 550 (FIM checksum changed) is 612, rule 510 (rootcheck) is 350.
- Design note: any "level 12 and above pushes to ntfy" tier must exclude the vulnerability detector rules. Rule 23506 is level 13 (352 alerts), so it would have pushed 352 times.

## Rule 550 under /etc/pve
- Volatile paths (.rrd, .version, .clusterlog, .vmlist, four lrm_status files) were 405 of 612. Most show 56 hits each, which is 4 nodes x 14 scans. Matches the 12 hour syscheck frequency (43200 s) confirmed in pve-env1 ossec.conf.
- Authkey files (authkey.pub, authkey.pub.old, priv/authkey.key) fire once per day per node. Verified: Proxmox rotates its auth key on a 24 hour cycle. Expected, but security relevant.
- /etc/apt/apt.conf.d/76pveproxy fired 28 times in the week (daily per node pattern). Cause unexplained.
- Real signal to keep: 511.conf edits (step-ca LXC work).

## Fix applied: suppression rule 100003
- local_rules.xml rule 100003 (April 25 decision, suppress everything under /etc/pve/) matched on `syscheck.path`. Wazuh docs name the decoded FIM field `file` and their custom FIM rule examples use `file`.
- Changed line 34 to `<field name="file">/etc/pve/</field>`. Manager restarted 2026-09-24 17:37:06 UTC. `wazuh-analysisd -t` exit 0.
- Agent on pve-env1 restarted. Scan on start ran and ended 17:39:38 UTC (agent log is in HDT on that node). Zero rule 550 alerts from pve-env1 after the restart. Before the change it averaged about 15 `/etc/pve` alerts a day.
- Verified on pve-env1 only. Other three nodes should be checked after their next 12 hour scan.
- No backup was taken. To revert, swap `file` back to `syscheck.path` on line 34 and restart the manager.
- Hypothesis, not proven: with `syscheck.path` the rule may never have matched properly. The April zero-count check ran right after a restart, before a scheduled scan could produce anything.

## Context that changed my reading
- Cluster was deliberately shut off Aug 17 (approx) to Sep 13. June to August data only covers pve-env2, pve-gateway and the VMs on them. That window is not a normal baseline.
- Manager upgrade history: 4.14.4 to 4.14.5 on Apr 25, 2026. No Wazuh package changes since, so the version is not the Sep change.
- pve-env1 clock runs on HDT, not UTC. pve-services runs on CDT. wazuh-manager runs UTC. This mismatch cost real time during the rule 100004 fix below, converting timestamps by hand to confirm scans ran after each restart.

## Fix applied: suppression rule 100004 (rootcheck false positive)
- Rule 510 rootcheck flagged 6 setuid binaries (chfn, chsh, passwd, in both /bin and /usr/bin) as "Trojaned version of file" on all 4 Proxmox nodes, every 12 hour scan. 336 of the 350 rule 510 alerts in the week were these 6 files.
- Confirmed false positive: `dpkg --verify` on the `passwd` package showed no checksum mismatch on pve-services. The only string in `/usr/bin/chsh` matching rootcheck's generic trojan regex was `/dev/n` (from `/dev/null`), present in nearly any compiled binary.
- Took 3 attempts to get the suppression rule to actually match (see [[wazuh-rootcheck-setuid-suppression]] for full reasoning on why each failed):
  1. Bare `<match>` with no type: defaults to literal string search, `$` and `|` treated as literal text. Didn't match.
  2. `type="osregex"`: Wazuh's own regex dialect, syntax valid, still didn't match. Cause not confirmed.
  3. `type="pcre2"` with `$` anchor: right engine, wrong anchor. The filename in `full_log` is followed by a closing quote, not end of line. Fixed by anchoring to the quote: `bin/passwd'` instead of `bin/passwd$`.
- Final working rule, added as 100004 in local_rules.xml, same file as 100003. Manager restarted 2026-09-25 01:10:50 UTC.
- Verified on pve-services only: agent restarted, rootcheck scan completed 01:12:43 UTC, 0 rule 510 hits for the 6 binaries after that. pve-env1, pve-env2, pve-gateway not yet independently confirmed, but run the same ruleset from the same manager.

## Open
- Vulnerability detector burst cause: `uname -r` and apt history on services-host and soc-stack, plus one expanded rule 23506 event (checking for a false positive on old CVEs against the 6.8 kernel). Never got to this.
- Decision pending: keep blanket /etc/pve suppression (current, matches April) or narrow it to the 8 volatile paths so authkey, lxc and qemu-server config changes stay visible. A narrow regex rule is drafted but untested.
- Confirm rule 100004 on pve-env1, pve-env2, pve-gateway after their next scan.
- Rules 5718, 5503, 5557: three denied-user SSH attempts and PAM/password failures on pve-env1. Source IP and user not checked.
- Rule 19011 (CIS sshd GSSAPIAuthentication) on admin-Standard-PC-Q35-ICH9-2009: 13 alerts in a 42 minute window starting 13:00 UTC today, new agent. Still on the default hostname.
- wazuh-analysisd warns that `if_sid` cannot be overwritten for rules 23502, 23508 and 19010. Not confirmed whether pre-existing.
- Tiering design (instant ntfy, daily digest, log only) not built yet.
- Standardize node clocks to UTC, or document each node's offset somewhere, so this stops being a manual conversion every fix.

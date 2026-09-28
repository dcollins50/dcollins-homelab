---
name: wazuh-rootcheck-setuid-suppression
type: decision
status: live
---

Suppress rootcheck rule 510 false positives on 6 setuid binaries across the Proxmox nodes, via [[SOC-Stack]].

## Decision
Added rule 100004 to `local_rules.xml` on wazuh-manager, suppressing rootcheck's generic trojan signature match on `/bin/chfn`, `/bin/chsh`, `/bin/passwd`, `/usr/bin/chfn`, `/usr/bin/chsh`, `/usr/bin/passwd`.

```xml
<rule id="100004" level="0">
  <if_sid>510</if_sid>
  <match type="pcre2">bin/chfn'|bin/chsh'|bin/passwd'</match>
  <description>FIM: rootcheck false positive on passwd/chsh/chfn setuid binaries, generic /dev/null signature match</description>
</rule>
```

## Reasoning
350 rootcheck alerts in 7 days, 336 of which were these 6 binaries on the 4 Proxmox nodes, firing identically on every 12 hour scan (14 scans x 6 files x 4 nodes = 336). Confirmed false positive: `dpkg --verify` on the `passwd` package showed no checksum mismatch, and the only string in `/usr/bin/chsh` matching rootcheck's generic trojan regex was `/dev/n` from `/dev/null`, a string present in nearly any compiled binary. Wazuh's own community support has documented the same false positive pattern on other binaries (e.g. `/usr/bin/diff`) with the same generic signature.

File integrity monitoring (rule 550) still covers `/bin`, `/usr/bin`, `/sbin`, `/usr/sbin` on all 4 nodes, so an actual binary replacement would still be caught.

## What was ruled out
- **Bare `<match>` with no type attribute** (first attempt): defaults to a literal string search (`sregex`), not regex. `$` and `|` were treated as literal characters, so the pattern never matched anything in `full_log`.
- **`type="osregex"`** (second attempt): Wazuh's own non-standard regex dialect. Syntax test passed and rule loaded, but still didn't match. Exact cause not confirmed, suspected to be non-standard handling of `$` as an anchor, but no citation found.
- **`type="pcre2"` with `$` anchor** (third attempt): correct engine, wrong anchor. The filename in `full_log` (e.g. `/bin/passwd`) is followed by a closing quote and more text (`' detected. Signature used...`), not the end of the line, so `bin/passwd$` never matched. Final fix anchored to the quote character instead: `bin/passwd'`.

## Verification
Confirmed on all 4 Proxmox nodes. pve-services confirmed same session (0 hits after agent restart + rootcheck scan). pve-env1, pve-env2, and pve-gateway confirmed the next day via the same method, see [[session-2026-09-25-rule-100004-confirmation]]: 0 hits on all three.

## Related
- Same session also fixed [[SOC-Stack]] rule 100003 (`/etc/pve` FIM suppression), which had a similar field-name mismatch (`syscheck.path` vs `file`).
- All 4 Proxmox nodes run on local time (pve-env1: HDT, pve-services: CDT, others unconfirmed) while wazuh-manager runs UTC. This cost significant time converting timestamps to confirm scans ran after each fix. Standardized to UTC across most of the fleet as of Sep 25, 2026, see [[standardize-node-clocks-utc]].
- Full session detail in [[session-2026-09-24-alert-engineering-baseline]].
- General procedure for this kind of fix, extracted afterward: [[wazuh-add-custom-rule]], [[wazuh-restart-agent-confirm-scan]].

---
name: wazuh-ignore-rootcheck-path
type: runbook
tool: wazuh
---

Suppress a rootcheck false positive at the agent config level, via an `<ignore>` path in the `<rootcheck>` block. Different from [[wazuh-add-custom-rule]] (which suppresses at the manager, in `local_rules.xml`, fleet-wide or scoped by match condition): this method excludes a specific path from rootcheck scanning on one agent only. Written up from the Jetson `/dev/mqueue` false positive investigated in an April 2026 session.

## When to use this vs. a manager-side suppression rule

- **This method (agent-side ignore)**: the false positive is tied to a specific path on a specific agent that will never be legitimate signal there (e.g. hardware/platform-specific boot artifacts). Simpler, doesn't touch `local_rules.xml`.
- **[[wazuh-add-custom-rule]] (manager-side)**: the false positive pattern could appear anywhere, needs to be suppressed fleet-wide, or needs conditional logic (frequency thresholds, matching on multiple fields). Rules 100003 and 100004 (see [[wazuh-rootcheck-setuid-suppression]]) both used this route instead, because the false positive was the same 6 binaries across all 4 Proxmox nodes, not a single-agent path issue.

## Example case: Jetson `/dev/mqueue` rootcheck false positive

Rule 521 ("Possible kernel level rootkit") fired 588 times on the Jetson from POSIX message queues in `/dev/mqueue/` created at boot by NVIDIA's NvSciBuf/NvSciStream/NvSciSync/NvMap IPC framework (L4T platform init, not a running process). Confirmed false positive via: identical creation timestamps across all the flagged files (all boot-time, not ongoing activity), `fuser` returning no owning process, and no matching userspace process in `pgrep`.

## Procedure

1. Confirm it's actually a false positive first. At minimum: check file timestamps for a pattern (all-identical suggests boot-time creation, not attacker activity), and check whether any process currently owns the flagged file/queue:
   ```
   fuser <path>
   pgrep -f <suspected-process-name>
   ```

2. On the affected agent, edit `/var/ossec/etc/ossec.conf` and add the path inside the existing `<rootcheck>` block:
   ```xml
   <rootcheck>
     <ignore>/dev/mqueue</ignore>
   </rootcheck>
   ```
   Check for existing `<ignore>` entries first; some agent configs already ship with common ones baked in (the Jetson's had `/var/lib/containerd` and `/var/lib/docker/overlay2` pre-existing from whatever baseline it was provisioned from).

3. Restart the agent:
   ```
   sudo systemctl restart wazuh-agent
   ```

4. Confirm the fix. Rootcheck runs on a 12-hour cycle by default, waiting for the natural cycle means no confirmation for up to 12 hours. Force an immediate scan instead, from [[wazuh-manager]] (not the agent):
   ```
   sudo /var/ossec/bin/agent_control -r -a
   ```
   Note this forces a rootcheck on every connected agent, not just the one being fixed, harmless but worth knowing if timing matters elsewhere.

5. Check Kibana for new hits on the rule from that agent after the forced-scan timestamp, per [[wazuh-restart-agent-confirm-scan]]. Existing alerts already in the index stay there, expected, only new occurrences should stop.

## Related
- [[wazuh-add-custom-rule]]
- [[wazuh-restart-agent-confirm-scan]]
- [[wazuh-rootcheck-setuid-suppression]]
- [[jetson-orin-nano]]

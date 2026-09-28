---
name: wazuh-restart-agent-confirm-scan
type: runbook
tool: wazuh
---

Trigger a rootcheck/FIM scan on a Wazuh agent (or the whole fleet at once) and confirm a specific alert rule has cleared. Built from actually doing this across all 4 Proxmox nodes on Sep 24/25, 2026 (see [[session-2026-09-24-alert-engineering-baseline]] and [[session-2026-09-25-rule-100004-confirmation]]).

## When to use this

After adding or editing a suppression/threshold rule in `local_rules.xml` and restarting [[wazuh-manager]], to confirm the change actually cleared the target alert on a given agent, rather than assuming a config reload worked.

## Procedure

There are two ways to trigger the scan. Pick based on scope.

**Option A — force a scan on every agent at once, from the manager.** Cleaner when confirming a fleet-wide rule change (this is what should have been used for the Sep 25 four-node rule 100004 confirmation instead of restarting each agent by hand):
```
sudo /var/ossec/bin/agent_control -r -a
```
Run this on [[wazuh-manager]] itself, not on the individual agents. Triggers an immediate rootcheck on every connected agent simultaneously.

**Option B — restart a single agent**, when only one host needs confirming:

1. Restart the agent:
   ```
   sudo systemctl restart wazuh-agent
   ```

2. Watch the agent log for the scan to actually complete, don't assume it ran just because the service restarted:
   ```
   sudo tail -f /var/ossec/logs/ossec.log
   ```
   Look for a line like `rootcheck: INFO: Ending rootcheck scan.` (for rootcheck-related rules) or `wazuh-syscheckd: INFO: (6009): File integrity monitoring scan ended.` (for FIM/rule 550-family rules). Restarting the agent triggers scan-on-start, no need to wait for the scheduled interval.

3. Note the completion timestamp and convert it to UTC. **Do not assume the node is on UTC or any particular offset** — check first:
   ```
   date && date -u
   ```
   Nodes in this homelab have been found on at least 3 different local timezones (HDT, CDT) despite the manager and Kibana being UTC-native. See [[standardize-node-clocks-utc]] for the fix in progress; until every node is confirmed migrated, keep checking rather than assuming.

4. In Kibana Discover, data view `wazuh-alerts-4.x-*`, set the time range to start at the converted UTC timestamp and query:
   ```
   agent.name: "<node-name>" and rule.id: "<rule-id>"
   ```
   Zero hits after that timestamp confirms the fix. Any hits: check whether they're actually the specific finding you suppressed or a different finding under the same rule ID (rule 510 rootcheck, for example, covers far more than any one suppression target).

## Notes

- Restarting the agent does not restart the manager or reload `local_rules.xml` — if the rule itself was just added, the manager needs its own restart first (see [[wazuh-add-custom-rule]]).
- Option A triggers the scan fleet-wide in one command but still requires per-agent Kibana verification afterward (step 4 above, run once per node/rule combination) — there's no single query that confirms every agent clean without checking `agent.name` individually, since a missing agent or a still-hitting agent both just look like "fewer total hits," not a clear per-node pass/fail.

## Related
- [[wazuh-manager]]
- [[wazuh-add-custom-rule]]
- [[session-2026-09-24-alert-engineering-baseline]]
- [[session-2026-09-25-rule-100004-confirmation]]

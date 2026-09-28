---
name: wazuh-add-custom-rule
type: runbook
tool: wazuh
---

Write, validate, and deploy a custom Wazuh rule (suppress, threshold, or a scoped match) on [[wazuh-manager]]. Extracted and corrected from the walkthrough in `Projects/soc-stack-buildout/soc-phase2-tuning.md`, which documents the concept well but has one outdated command (see Validation below). Confirmed against real usage twice on Sep 24/25, 2026 (rules 100003, 100004).

## Background

Custom rules live in `/var/ossec/etc/rules/local_rules.xml`, inside a `<group name="local,">` block. Never edit the built-in rules under `/var/ossec/ruleset/rules/` directly, they get overwritten on update. Custom rule IDs must be in the 100000–120000 range.

When a custom rule shares a built-in rule's ID (with `overwrite="yes"`), the custom rule wins. This is the mechanism for suppressing or modifying built-in behavior fleet-wide. A new rule ID with a `match`/`field` condition instead scopes the change to a specific agent, path, or pattern rather than suppressing the built-in rule for everyone.

## The three tuning decisions

- **Suppress**: alert level 0. Use when the underlying event has no security value in this environment.
- **Threshold**: `frequency`/`timeframe` attributes. Use when only bulk occurrence matters (e.g. N auth failures in 60s).
- **Accept**: no rule change, just document the decision. Use for genuine signal, even if high-volume.

## Procedure

1. Open the custom rules file:
   ```
   sudo nano /var/ossec/etc/rules/local_rules.xml
   ```

2. Add the rule inside the existing `<group name="local,">...</group>` block. Example, matching on a decoded field rather than the older `syscheck.path`:
   ```xml
   <rule id="100003" level="0" overwrite="yes">
     <if_sid>550</if_sid>
     <field name="file">/etc/pve/</field>
     <description>Suppress FIM under /etc/pve (expected Proxmox internal churn)</description>
   </rule>
   ```
   Confirmed field name matters: `syscheck.path` did not reliably match in practice (see [[session-2026-09-24-alert-engineering-baseline]] for the debugging trail), Wazuh's decoded FIM field is `file`.

3. **Validate before restarting.** This is the one place the Phase 2 doc's original guidance is outdated:
   ```
   sudo /var/ossec/bin/wazuh-analysisd -t
   ```
   Exit code 0 means clean. This is the current, correct command for Wazuh 4.x, confirmed working during the Sep 24 session. The Phase 2 doc's `ossec-logtest -t` is the Wazuh 3.x-era tool name; in 4.x, `wazuh-logtest` still exists but is an interactive single-log-line tester, not a full-config validator with a reliable exit code for this purpose. Use `wazuh-analysisd -t`.

4. If validation fails, check XML syntax first:
   ```
   sudo xmllint --noout /var/ossec/etc/rules/local_rules.xml
   ```
   Common mistakes: unclosed tags, unescaped `&`, duplicate rule IDs across custom rule files.

5. Restart the manager:
   ```
   sudo systemctl restart wazuh-manager
   ```

6. Confirm the rule loaded:
   ```
   sudo grep 'Loaded rule' /var/ossec/logs/ossec.log | grep '<rule-id>'
   ```

7. Confirm the actual effect on each affected agent using [[wazuh-restart-agent-confirm-scan]].

## Notes

- Manager restart timestamp is in the manager's own local time (confirmed UTC as of Sep 25, 2026, see [[standardize-node-clocks-utc]]), but the agents you're confirming against may not be, check each one.
- A suppression via `overwrite="yes"` on an existing rule ID (like 100003/100004 above) is fleet-wide. To scope to one agent or path instead of suppressing globally, use a new rule ID with `<if_sid>`, `<match>`/`<field>`, and optionally `<same_source_ip/>` for threshold rules, rather than `overwrite`.
- No backup was taken before the Sep 24 rule 100003 edit. To revert any change here, keep the pre-edit line commented or noted somewhere before saving.

## Related
- [[wazuh-manager]]
- [[wazuh-restart-agent-confirm-scan]]
- [[session-2026-09-24-alert-engineering-baseline]]

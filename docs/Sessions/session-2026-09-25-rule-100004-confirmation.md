---
name: session-2026-09-25-rule-100004-confirmation
type: session
date: 2026-09-25
---

Follow-up from [[session-2026-09-24-alert-engineering-baseline]]. Goal: confirm Wazuh suppression rule 100004 (rootcheck setuid false positive) cleared on the three nodes not yet checked.

## Node confirmations

For each node: `sudo systemctl restart wazuh-agent`, tail `/var/ossec/logs/ossec.log` for rootcheck completion, convert local timestamp to UTC, then query Kibana Discover (`wazuh-alerts-4.x-*`) for `agent.name: "<node>" and rule.id: "510"` starting at that UTC time.

- **pve-env1**: rootcheck ended 13:20:20 local (HDT) / 22:20:20 UTC. Zero rule 510 hits over the full 24h window.
- **pve-env2**: rootcheck ended 13:21:53 local (HDT) / 22:21:53 UTC. Zero rule 510 hits.
- **pve-gateway**: rootcheck ended 17:25:57 local (CDT) / 22:25:57 UTC. Zero rule 510 hits.

All three clean, matching pve-services from the Sep 24/25 session. Rule 100004 confirmed fleet-wide across all four Proxmox nodes.

## Node clock offsets found

- pve-env1: HDT
- pve-env2: HDT
- pve-gateway: CDT
- pve-services: CDT (per Sep 24 session)
- wazuh-manager: UTC

This is the third time a node's local timezone has had to be manually converted to UTC to confirm a fix. The Sep 24 session flagged "standardize node clocks to UTC, or document each node's offset somewhere" as an open item; still open.

## Related
- [[session-2026-09-24-alert-engineering-baseline]]

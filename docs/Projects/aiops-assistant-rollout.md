---
name: aiops-assistant-rollout
type: project
status: planned
---

Rollout plan for the AI ops assistant that runs on the [[jetson-orin-nano]] on [[VLAN11]]. Planned Sep 23, 2026. Nothing here is started, and everything is proposed, not agreed, unless noted. The network build is in [[aiops-vlan-build]]. The decision behind the VLAN is in [[aiops-vlan]].

## Scope (Daniel, Sep 23, 2026)
Watch, report, diagnose, attempt easy non-destructive fixes, or alert and escalate to Daniel or whoever runs the stack. It is only Daniel for now. If provisioned for another user or org, alert routing must be configurable per recipient.

## Principles (proposed)
- The model classifies and explains. It never writes commands. Pre-written, tested playbooks in [[soar-host]] (Shuffle) do any acting, from an allowlist of actions and targets.
- Hostnames and parameters come from alert metadata, never from model text.
- Alerts never depend on the AI. If the jetson is down, slow or returns invalid output, the raw alert still goes out, marked as having no AI analysis.
- Model output must match a fixed schema, or it is discarded.

## Phase 0: deterministic detection (no model)
Plain checks for the failures the postmortems found undetected for days: node quorum state, no new documents from a log source for N minutes, wazuh-manager disk usage, Elasticsearch shard usage, and a VM that should be running but is stopped. These are the highest-value alerts and need no LLM.

## Phase 1: shadow mode
The assistant receives alert context and writes recommendations to a log only. No notifications and no actions. Daniel compares them with what he would have done. Start on the Uptime Kuma path, which already works through Shuffle's debounce workflow ([[soar-host]]). Wazuh alerts join once the Wazuh to Shuffle trigger exists.

## Phase 2: alerting through ntfy
The assistant starts talking to Daniel.

Flow: alert, then Shuffle debounce (the pattern already proven on the Kuma path), then the jetson classifies and explains, then Shuffle validates the output against the schema, then routing, then an [[ntfy]] message and a note on the case in [[pve-iris]].

Message contents: what happened and where (from metadata), a short AI summary labelled as AI-generated, likely cause, a suggested next step chosen from a fixed list, confidence, and links to the case and Kibana. Severity maps to ntfy priority.

Prerequisites:
- [[VLAN11]] built and the jetson serving Ollama on a chosen port, with a rule from [[soar-host]] to that port.
- Wazuh to Shuffle trigger. The threshold is still undecided: severity or specific rule IDs. See [SOC Alert Pipeline](../SOPs/sop-soc-alert-pipeline.md). The backlog mentions level 12 and above.
- A firewall path from [[soar-host]] to ntfy. Not checked.
- A dedicated topic and write-only publisher account for the assistant, following the existing `kuma-alerts` and `kuma-publisher` pattern (deny-all default).
- Phase 1 results good enough to trust.

Design elements:
- Routing is data, not workflow logic: a small config mapping severity and asset to channel, priority, recipient, quiet hours and escalation timer. Provisioning for another org means editing config.
- Noise control: debounce, dedupe by rule and host over a window, low severity into a daily digest.
- Escalation: unacknowledged after N minutes goes to a second channel or person.
- Feedback: ntfy action buttons ("useful" and "wrong") post back to Shuffle. This is the ntfy action-button stretch goal already on the backlog. It builds the accuracy record needed to decide on phase 3.

Open decisions: the Wazuh trigger threshold, notification channels, quiet hours, escalation timing, and who else, if anyone, receives alerts.

## Phase 3: fixes
Start with human approval through an ntfy action button that triggers the playbook. Only later, and only for the safest actions, allow automatic runs, with per-host and per-hour limits, a kill switch and an audit trail in Elastic.

Candidate allowlist from past incidents: start a stopped VM from a listed set, restart a named service once, restart a named container once.

Never automatic: log truncation, index deletion, corosync edits, and firewall, DNS or certificate changes.

Fixes need a path into production. SSH goes through the bastion's two-MFA flow, which a service cannot complete, so fixes use API calls with narrowly scoped tokens.

## Related
- [[aiops-vlan-build]]
- [[aiops-vlan]]
- [[jetson-orin-nano]]
- [[soar-host]]
- [[ntfy]]
- [[dhcp-dns-rollout]]

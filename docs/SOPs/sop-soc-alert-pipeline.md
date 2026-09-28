---
name: sop-soc-alert-pipeline
type: sop
status: in-progress
---

# SOP: SOC Alert Pipeline (Detection → SOAR → Case → Notification)

**Status:** Partially built. Do not treat this as a working end-to-end pipeline yet — see Status by Stage below.
**Objective:** Get a real detection event from Wazuh/Suricata through to a human getting notified and a case existing to track it, without every DOWN event or noisy alert generating a ticket.

---

## Why this is an SOP and not a runbook

This spans four separate tools with independent lifecycles ([[wazuh-manager]], [[soar-host]] running Shuffle, [[pve-iris]] running DFIR-IRIS, and [[ntfy]] behind [[authentik]]), and the actual value is in how they hand off to each other, not any one tool's configuration.

## Intended flow

```
Wazuh alert / Uptime Kuma DOWN event
        │
        ▼
Shuffle SOAR (soar-host, VM 602) — debounce + routing
        │
        ▼
DFIR-IRIS (pve-iris, VM 604) — case created
        │
        ▼
ntfy — human notified
```

## Status by stage

**Uptime Kuma → Shuffle → IRIS (built, Sep 19, 2026):**
Debounce workflow confirmed working: after Kuma's own retries produce a confirmed-DOWN event, Shuffle waits 60 seconds, rechecks, and only creates an IRIS ticket if the service is still down. This was a deliberate design choice — reducing false-positive ticket generation was the top priority for this build, since a ticket on every transient blip defeats the point of having tickets at all.

**Wazuh alerts → Shuffle (not confirmed built):**
The debounce workflow described above is triggered by Uptime Kuma, not by Wazuh alert severity. Nothing in the current documentation confirms Wazuh alerts feed into a Shuffle workflow yet. This is the actual gap in the pipeline — Wazuh alerts land in Kibana dashboards (see [[SOC-Stack]]) but there's no confirmed automated handoff from "Wazuh fired a high-severity alert" to "a SOAR workflow reacted to it."

**SOAR/IRIS → ntfy (not built):**
[[ntfy]] is deployed and reachable, and sits behind Authentik forward-auth with Uptime Kuma's own native ntfy notification support already usable directly. But a Wazuh-to-ntfy integration specifically, and automated ntfy push for any root/admin/superuser-level sign-in across the homelab, are both listed as planned, not built.

## Firewall status

[[soar-host]] and [[pve-iris]] don't yet have dedicated firewall rules — both currently rely on the general SOC net (VLAN10) outbound policy. This is a known open item, not an oversight.

## What to build next, in dependency order

1. Decide what Wazuh alert condition should trigger a Shuffle workflow (severity threshold? specific rule IDs?) — this hasn't been decided yet, and deciding it is a prerequisite to building it, not a technical step.
2. Wire that trigger into a new Shuffle workflow, following the same debounce-before-ticket philosophy already proven out on the Kuma path.
3. Wire Wazuh-to-ntfy and the admin/root sign-in notification separately — these don't depend on the Shuffle work above and could be done first if they're higher priority.
4. Scope dedicated firewall rules for soar-host and pve-iris instead of relying on the general SOC net policy.

## Related Documentation

- [[wazuh-manager]]
- [[soar-host]]
- [[pve-iris]]
- [[ntfy]]
- [[authentik]]
- [[SOC-Stack]]
- [[opnsense-agentless-ssh-monitoring-setup|Agentless SSH Monitoring Setup]]
- [[elk-import-configure-wazuh-dashboards|Import/Configure Wazuh Dashboards]]

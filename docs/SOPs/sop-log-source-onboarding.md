---
name: sop-log-source-onboarding
type: sop
status: complete
---

# SOP: Onboard a New Log Source (End to End)

**Status:** Every stage below already exists as its own runbook. This SOP just sequences them — nothing here is new procedure, it's the order that avoids the failure modes already hit in this environment.
**Objective:** Get a new host or service's logs flowing into [[SOC-Stack]] and visible in Kibana, without hitting the two failure modes that have already happened here: a silently-blocked firewall path, and a TLS output block missing on a new pipeline.

---

## Why this is an SOP and not just "read the ELK runbook"

<!-- redacted: operational security detail redacted for public repo -->

## Order matters — do not skip ahead

### 1. Firewall path first

Confirm or add the firewall rule permitting the new source's VLAN to reach Logstash's ingest port(s) on [[VLAN10]]. Check whether the source fits an existing port-group alias before creating a one-off rule.

→ [[opnsense-add-firewall-rule|Add a Firewall Rule]]

<!-- redacted: operational security detail redacted for public repo -->

### 2. TLS, if this is a new pipeline file

If the source needs its own new Logstash output block (rather than extending an existing pipeline), issue a cert from the Intermediate CA and configure the output for TLS now, not after the pipeline is already running unencrypted.

→ [[elk-configure-tls-elasticsearch-kibana|Configure TLS (Elasticsearch/Kibana)]]

### 3. Build or extend the Logstash pipeline

Input/filter/output blocks on [[soc-stack-vm]]. Watch specifically for noisy, high-frequency log types — this environment has already hit a grok-filter stall from an unexpectedly chatty source once; add a drop filter for known-noisy message patterns before it reaches the main parsing logic.

→ [[elk-add-log-source-pipeline|Add a Log Source / Pipeline]]

### 4. Retention policy

Don't let the new index inherit no ILM policy by default — this is exactly what caused the `cluster.max_shards_per_node` outage in May 2026. New sources default to the 90-day general policy unless there's a specific reason (like `wazuh-alerts`) to extend it.

→ [[elk-manage-ilm-retention|Manage ILM Retention]]

### 5. Kibana visibility

Confirm/create an index pattern, then build whatever Lens visualizations or dashboard panels the source needs, following the standing X-axis-is-dimension/Y-axis-is-count convention.

→ [[elk-build-kibana-lens-visualization|Build a Kibana Lens Visualization]]

### 6. If something doesn't show up

Work the diagnostic order in the troubleshooting runbook — it's ordered specifically to distinguish a capacity problem from a firewall problem from a pipeline problem, rather than guessing.

→ [[elk-troubleshoot-log-shipping-stopped|Troubleshoot Log Shipping Stopped]]

## Related Documentation

- [[SOC-Stack]]
- [[soc-stack-vm]]
- [[VLAN10]]
- [[PKI]]

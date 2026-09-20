# Incident: Elastic Log Shipping Outage

**Date:** 2026-09-13 – 2026-09-14
**Systems affected:** soc-stack (VM 600, 10.0.10.10), wazuh-manager (VM 601, 10.0.10.11), Jetson Orin Nano (10.99.0.100, VLAN40)
**Status:** Resolved

## Summary

Following a cluster shutdown of roughly 1-2 months, log shipping into Elasticsearch broke across multiple sources. Three separate, mostly unrelated root causes were found and fixed:

1. `wazuh-manager` disk filled to 100% from an unrotated alert log
2. Elasticsearch hit its default shard ceiling from months of unmanaged daily indices, blocking all new index creation
3. A pre-existing OPNSense firewall rule silently blocked the Jetson's log shipping traffic, unrelated to the outage itself

## Timeline

- **Aug 20, 2026** — Homelab cluster taken offline (~1-2 months).
- **~Aug 20** — Jetson's Filebeat begins failing to reach soc-stack on port 5044 (firewall issue, see Root Cause 3). This predates the shutdown and was independent of it.
- **Sep 13** — Cluster brought back online. Wazuh alert volume spikes; `wazuh-manager` root filesystem fills to 100%.
- **Sep 13 (evening)** — Diagnosed and fixed the disk-full condition on `wazuh-manager`.
- **Sep 14 (morning)** — Discovered `proxmox-logs-*`, `opnsense-logs-*`, etc. had no new documents since ~19:00 the prior evening. Diagnosed as an Elasticsearch shard limit.
- **Sep 14** — Fixed shard limit, cleaned up stale indices, implemented ILM retention policies.
- **Sep 14** — Discovered Jetson logs still not flowing after the above fixes; traced to a firewall rule, unrelated to the other two issues.

## Root Cause 1: wazuh-manager disk full

`/var/ossec/logs/alerts/2026/Aug/ossec-alerts-20.json` and its companion `.log` file never rotated when the cluster went down mid-write on Aug 20. An internal authentication-failure loop on `wazuh-manager` itself (Wazuh rule 2501, `no-srcip`, not an external attack) wrote continuously into that single file, growing it to 62G and filling the 96G root volume to 100%. This broke Filebeat's ability to write its registry checkpoint, halting delivery of Wazuh alerts to Elasticsearch.

**Fix:**
```
truncate -s 0 /var/ossec/logs/alerts/2026/Aug/ossec-alerts-20.json
truncate -s 0 /var/ossec/logs/alerts/2026/Aug/ossec-alerts-20.log
systemctl restart filebeat
```
Freed the volume from 100% used to ~38% used (57G available).

## Root Cause 2: Elasticsearch shard ceiling

No ILM (Index Lifecycle Management) policy had ever been configured for any log source. Each source (`proxmox-logs`, `opnsense-logs`, `heimdall-logs`, `suricata-logs`, `jetson-logs`, `wazuh-alerts`, `wazuh-archives`) creates a new daily index, and each new index consumes at least one shard. Over ~9 months of operation, this accumulated to 938 indices / 997 active shards, against Elasticsearch's default `cluster.max_shards_per_node` limit of 1000. Once the limit was hit, every new index creation attempt (i.e., every new day's logs, across all sources) failed with a `validation_exception`, silently halting ingestion cluster-wide.

**Immediate fix:**
- Raised `cluster.max_shards_per_node` to 2000 as headroom.
- Deleted 596 indices older than a 90-day cutoff (June 16, 2026), bringing shard count from 997 down to ~379.

**Long-term fix — ILM retention policies:**

| Policy | Retention | Applies to |
|---|---|---|
| `logs-90-day-retention` | 90 days | proxmox-logs, opnsense-logs, heimdall-logs, suricata-logs, jetson-logs, wazuh-archives |
| `wazuh-alerts-365-day-retention` | 365 days | wazuh-alerts (kept longer — lower volume, higher-signal incident data) |

Attached via:
- Composable template `single-node-defaults` (covers the five general Logstash-fed sources) — added `index.lifecycle.name`.
- New templates `wazuh-alerts` and `wazuh-archives`, split out from the previously combined legacy `wazuh` template so each could carry its own ILM policy. The old combined `wazuh` template was deleted.

**Note:** Wazuh manager may reassert its own default template on service restart/upgrade, which could silently revert this split. Worth re-checking the template split after any Wazuh manager restart or upgrade.

Both new templates and today's already-existing indices were backfilled with the correct `index.lifecycle.name` so nothing was orphaned from the policy.

## Root Cause 3: OPNSense blocking Jetson log traffic

Independent of the above, the Jetson's Filebeat (VLAN40, `10.99.0.100`) had been unable to reach soc-stack's Logstash beats input on port 5044 since Filebeat was last restarted (Aug 20), with connection attempts silently timing out (46,000+ failed reconnect attempts).

Diagnosis ruled out, in order: Logstash health (fine, confirmed listening), routing/gateway reachability from the Jetson (fine), and finally the OPNSense ruleset on `VLAN40_SecurityLab1`, which had an unlogged **"Block VLAN40 to internal VLANs"** rule with no carve-out for log shipping (only Wazuh agent ports 1514-1515 were allowed through, via an existing rule). Because the block rule had logging disabled, it never appeared in the OPNSense filter log, which delayed the diagnosis.

**Fix:** Added two reusable (non-VLAN-specific) aliases and one new allow rule above the block rule on `VLAN40_SecurityLab1`:

- Alias `SOC_Stack_Ingest` (host) → `10.0.10.10`
- Alias `SOC_Stack_Log_Ports` (port list) → `5044, 5045, 5144, 5145`
- Rule: Pass, VLAN40_SecurityLab1 net → `SOC_Stack_Ingest` on `SOC_Stack_Log_Ports`, logging enabled, positioned above the internal-VLAN block rule.

## Verification

- `curl -sk -u elastic:<pw> https://localhost:9200/_cluster/health | jq '.active_shards'` — confirmed drop from 997 to ~379 shards.
- Logstash logs (`journalctl -u logstash`) confirmed clean writes with no further `validation_exception` errors.
- Kibana Discover confirmed live data resuming across all sources.
- `nc -zv 10.0.10.10 5044` from the Jetson confirmed connectivity after the firewall fix; `journalctl -u filebeat` confirmed a clean reconnect.

## Follow-ups / open items

- [ ] Confirm the `wazuh-alerts` / `wazuh-archives` template split survives a Wazuh manager restart or upgrade.
- [ ] Revisit retention windows once the dedicated storage stack is built (90/365-day policy was chosen under current storage constraints).
- [ ] Consider a severity-based split for `wazuh-alerts` (e.g. `rule.level` ≥ 12 into a separate longer-retention index) — deferred, documented as a future project, not implemented today.
- [ ] Consider re-disabling logging on the new VLAN40 allow rule once confidence is established, or leave it on for ongoing visibility (currently left enabled).

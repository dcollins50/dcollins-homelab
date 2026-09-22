# Incident: Elasticsearch Shard Limit Exhaustion (INC-2026-001)

**Date:** 2026-05-19 (est. start) to 2026-05-22 (resolved)
**Systems affected:** soc-stack (VM 600, 10.0.10.10), Elasticsearch, Logstash
**Severity:** High. All Logstash-managed log ingestion stopped for ~3-4 days.
**Detected by:** Manual check. No new documents visible in Kibana since May 18.
**Status:** Resolved (recurred in September, see Related)

## Summary

All Logstash-managed pipelines stopped writing to Elasticsearch on or around May 19, 2026. Affected indices were `proxmox-logs-*`, `opnsense-logs-*`, `heimdall-logs-*`, `jetson-logs-*` and `suricata-logs-*`. Wazuh kept working the whole time. Sources kept sending to Logstash, but Elasticsearch rejected every new index.

## Timeline

- **Late Feb 2026** Daily Logstash indices begin accumulating across five sources, each with 1 primary and 1 replica shard.
- **~May 18-19** Total open shards crosses 1000. Elasticsearch starts rejecting new index creation with `validation_exception` (HTTP 400). Logstash logs the rejections at WARN and keeps retrying.
- **May 22** Gap noticed in Kibana. Index creation dates, Logstash service status and live Logstash errors checked. Shard limit identified.
- **May 22** Replicas set to 0 on all indices. Ingestion resumed within minutes.
- **May 22** Index template created to stop replicas on future indices. Took two attempts.
- **May 22** Ingestion verified on all sources except Jetson.

## Root Cause

Elasticsearch allows 1000 open shards per node by default (`cluster.max_shards_per_node`). soc-stack is a single-node cluster, so replica shards can never be assigned. They stayed unassigned (which is why every Logstash index had been yellow the entire time) but still counted against the limit. After roughly three months of daily indices across five sources, the count hit 1000.

Wazuh was not affected because its own index templates already set `number_of_replicas: 0`.

## Fix

1. Drop replicas on every existing index. This halved the shard count and unblocked index creation immediately.

```bash
curl -sk -u elastic:<password> -X PUT "https://10.0.10.10:9200/*/_settings" \
  -H "Content-Type: application/json" \
  -d '{"index": {"number_of_replicas": 0}}'
```

2. Create an index template, `single-node-defaults`, with `number_of_replicas: 0` for the five Logstash patterns.

The first attempt used `"index_patterns": ["*"]`. Elasticsearch rejected it because it overlapped with the Wazuh templates at priority 1 and the built-in `.monitoring-*` templates at priority 0. The working version uses priority 1 and the five explicit patterns: `proxmox-logs-*`, `opnsense-logs-*`, `heimdall-logs-*`, `jetson-logs-*`, `suricata-logs-*`.

## Verification

| Source | Index | Doc count at verification |
|-|-|-|
| proxmox-logs | 2026.05.22 | 820 |
| opnsense-logs | 2026.05.22 | 7,157 |
| heimdall-logs | 2026.05.22 | 57 |
| suricata-logs | 2026.05.22 | 4 |
| jetson-logs | 2026.05.22 | Not present (separate, pre-existing issue) |

`single-node-defaults` confirmed with `number_of_replicas: 0`.

## Detection Gap

Nothing alerted. The outage ran 3-4 days and was found only because the dashboards looked empty. There was no alert on Logstash write failures or on shard count.

## Lessons Learned

- On a single-node cluster, replicas should be 0 from day one. The default silently burns half the shard budget on shards that can never be assigned.
- Without a retention policy, daily indices will hit the shard ceiling again. Removing replicas only buys time.
- Pipeline liveness needs its own alert. A quiet dashboard is not proof that nothing is happening.

## Open items at close

| Item | Priority | What happened later |
|-|-|-|
| Jetson log gap (last index 2026.05.09) | Medium | Traced on Sept 14 to an OPNSense rule blocking Filebeat on port 5044. |
| Index Lifecycle Management | High | Deferred at the time. Not in place when the shard limit was hit again in September. ILM implemented Sept 14. |
| Alerting on pipeline failure / shard usage | High | No record in chat history of this being done. |

## Related

- [incident-review-sept14.md](incident-review-sept14.md): the same shard limit was hit again after the cluster came back from its August shutdown.

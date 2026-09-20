# Runbook: Troubleshoot Log Shipping Stopped (ELK)

| Field | Value |
| --- | --- |
| Applies to | Elasticsearch, Logstash, Filebeat, OPNSense (firewall) |
| Category | Troubleshooting |
| Author | Daniel Collins |

---

## When to use this

Use this when logs from a source stop appearing in Kibana, whether that's noticed via a dashboard going quiet or via an actual alerting gap. This environment has hit two genuinely different root causes for this before, a capacity problem and a firewall problem, so the diagnostic order below is built to distinguish between them quickly rather than guessing.

---

## Diagnostic order

Work through these in order, each step rules out a category of cause before moving to the next, don't skip ahead based on a hunch, the two real incidents here had non-obvious root causes that a narrower initial guess would have missed.

### 1. Is Elasticsearch itself healthy?

```bash
curl -s -u elastic:<password> "https://<elasticsearch-ip>:9200/_cluster/health?pretty" -k
```

Check status (green/yellow/red) and specifically check for shard limit issues:

```bash
curl -s -u elastic:<password> "https://<elasticsearch-ip>:9200/_cluster/settings?include_defaults=true" -k | grep max_shards_per_node
```

**This environment has hit `cluster.max_shards_per_node` before**, from unbounded index growth with no ILM policy in place. If the cluster is at or near the shard limit, new index creation (and therefore new writes) can fail cluster-wide, not just for one source. See [Manage ILM Retention](manage-ilm-retention.md) if this is the cause, this is a capacity problem, not a per-source problem.

### 2. Is the specific target host/service actually up and not disk-full?

```bash
df -h
systemctl status elasticsearch logstash
```

**Also hit before:** a Wazuh Manager VM going to 100% disk from an unrotated alerts log file that never rotated after the cluster had been down for a while. Disk exhaustion on any stack component can silently stop writes. Check disk space on soc-stack, wazuh-manager, and the affected source host, not just the one you suspect.

### 3. Is Logstash's pipeline for this source actually running and error-free?

```bash
journalctl -u logstash -n 100
```

Check for:
- Pipeline worker errors or stalls (a grok timeout from an unexpectedly large/malformed message has happened here before, see the noisy-source note in [Add a Log Source / Pipeline](add-log-source-pipeline.md))
- TLS/certificate errors (see [Configure TLS](configure-tls-elasticsearch-kibana.md) if so)
- Connection refused/timeout errors to Elasticsearch

### 4. Is the traffic actually reaching Logstash at all?

This is the step that's easy to skip if you're focused on Logstash/Elasticsearch internals, but **this has been the actual root cause here before** (the Jetson Filebeat outage): a firewall rule silently blocking the source VLAN's traffic to the SOC ingest port, with logging disabled on the blocking rule so it produced no visible trace.

1. Confirm the firewall rule exists and is actually permitting this specific source's traffic to the ingest port, don't just assume it does because it did once, check the live ruleset. See [Add a Firewall Rule](../opnsense/add-firewall-rule.md).
2. If a block rule with logging disabled is a possibility, consider temporarily enabling logging on the relevant block rule to confirm/deny this is the cause, rather than guessing.
3. Test connectivity directly from the source host:
   ```bash
   nc -zv <soc-stack-ip> <ingest-port>
   ```

### 5. Is the source itself actually generating/shipping logs?

Only after ruling out 1-4: check the source's own shipping agent (Filebeat, syslog config, etc.) is actually running and pointed at the correct destination.

```bash
journalctl -u filebeat -n 50
```

---

## Common mistakes

- **Jumping straight to "check Logstash" without first ruling out a cluster-wide capacity issue**, wastes time on a single-pipeline theory when the real cause affects everything.
- **Assuming the firewall is fine because it "was fine before."** A rule can silently stop matching after other rule changes, or have logging disabled specifically on the rule that matters, don't skip step 4 just because it seems unlikely.
- **Not checking disk space early.** It's a fast check and rules out an entire category of failure immediately.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Manage ILM Retention](manage-ilm-retention.md)
- [Add a Log Source / Pipeline](add-log-source-pipeline.md)
- [Configure TLS](configure-tls-elasticsearch-kibana.md)
- [Add a Firewall Rule](../opnsense/add-firewall-rule.md)

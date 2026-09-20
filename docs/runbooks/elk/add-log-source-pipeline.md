# Runbook: Add a Log Source / Pipeline (ELK)

| Field | Value |
| --- | --- |
| Applies to | Logstash, Elasticsearch, Kibana |
| Category | Log Ingestion |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new host, service, or appliance needs its logs flowing into the SOC stack, this environment already has separate pipelines per source type (Proxmox, Suricata, OPNSense, Heimdall, Jetson), so a new source usually means either a new pipeline file or a new input/filter block within an existing one, not a from-scratch Logstash setup.

---

## Before you start

- **Know the log format and transport.** Syslog? A log-shipping agent (Filebeat)? A structured format like EVE JSON? This decides the Logstash input plugin.
- **Know the destination index naming.** Follow the existing convention (`<source>-logs-*`) rather than inventing a new pattern.
- **Confirm the firewall path exists.** The source needs an explicit rule to reach Logstash's ingest port(s) on the SOC VLAN, this has been the actual root cause of "why isn't this showing up" more than once here (see [Troubleshoot Log Shipping Stopped](troubleshoot-log-shipping-stopped.md)), don't skip this and assume connectivity.

---

## Steps

### 1. Confirm/add the firewall rule

The new source's VLAN needs an allowed path to Logstash's ingest port(s) on the SOC VLAN. See [Add a Firewall Rule](../opnsense/add-firewall-rule.md). This environment has a dedicated alias grouping the standard SOC ingest ports, check whether the new source fits an existing alias/port group before creating a one-off rule.

### 2. Create or extend the Logstash pipeline

On soc-stack, in `/etc/logstash/conf.d/`:

- If this source fits logically with an existing pipeline file (e.g. another host type feeding the same general pipeline), add an input/filter/output block there.
- If it's a genuinely distinct source type, create a new `.conf` file following the naming and structure of the existing ones.

A pipeline generally needs:
- **Input** block: matching the transport (syslog listener, Beats input, etc.) and port
- **Filter** block: parsing (grok, JSON, etc.) to structure the incoming data, plus any noise-suppression needed (see the note below on drop filters)
- **Output** block: pointing at Elasticsearch, with the correct destination index pattern and using the internal PKI cert for TLS, see [Configure TLS](configure-tls-elasticsearch-kibana.md) if this is a new pipeline file that needs its own output block configured for TLS

### 3. Watch for noisy sources

This environment has hit this before: a source that logs high-frequency, low-value messages (e.g. periodic stats messages) can choke a grok filter or just flood the index with noise. If the new source has a similarly chatty log type, add a drop filter for the specific noisy message pattern before it reaches the main parsing logic, rather than parsing and indexing it and cleaning up after the fact.

### 4. Validate pipeline syntax before restarting

```bash
sudo -u logstash /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  --path.data /tmp/logstash-test \
  -f /etc/logstash/conf.d/<pipeline-file>.conf
```

Should return `Configuration OK`. Don't restart Logstash on unvalidated config, a syntax error can take down pipelines that were working fine.

### 5. Restart Logstash and verify

```bash
sudo systemctl restart logstash
```

- Check `/var/log/logstash/logstash-plain.log` for errors on startup, particularly TLS/cert errors if this pipeline is new.
- Confirm the new index is actually being created and receiving documents:
  ```bash
  curl -s -u elastic:<password> "https://<elasticsearch-ip>:9200/<new-index-pattern>-*/_count" -k
  ```
- In Kibana, create/confirm an index pattern for the new index if one doesn't already exist, so it's actually queryable.

### 6. Document it

Add the new source and its log flow to [SOC Stack](../../soc-stack.md)'s log flow section.

---

## Common mistakes

- **Forgetting the firewall rule** and spending time debugging Logstash when the traffic never arrives in the first place, check connectivity before diving into pipeline config.
- **Skipping config validation** before restarting Logstash, risking downtime for every pipeline, not just the new one.
- **Not accounting for a noisy log source up front**, leading to the same kind of grok-timeout pipeline stall this environment has already hit once.
- **Forgetting the TLS output config on a new pipeline file**, an easy thing to miss if copying an old pipeline that predates the TLS hardening work.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Configure TLS (Elasticsearch/Kibana)](configure-tls-elasticsearch-kibana.md)
- [Troubleshoot Log Shipping Stopped](troubleshoot-log-shipping-stopped.md)
- [Add a Firewall Rule](../opnsense/add-firewall-rule.md)

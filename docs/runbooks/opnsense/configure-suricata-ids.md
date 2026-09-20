# Runbook: Configure Suricata IDS (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense |
| Category | Intrusion Detection |
| Author | Daniel Collins |

---

## When to use this

Use this for initial Suricata setup, ruleset maintenance, and routine alert review. Suricata currently runs in detection-only (IDS) mode on the WAN interface, logging to EVE JSON and forwarding to the ELK stack for analysis in Kibana. It is not yet promoted to inline blocking (IPS) mode, see the note at the bottom for what that switch will involve when it happens.

---

## Initial setup (already done, documented for reference/rebuild)

1. **Services → Intrusion Detection → Administration.** Enable the IDS/IPS engine.
2. **Select the interface(s) to monitor.** Currently WAN only, monitoring inbound traffic at the perimeter rather than internal inter-VLAN traffic.
3. **Set mode to IDS (detection only), not IPS.** This is the deliberate current posture, not a placeholder, see the IPS note below for why.
4. **Select and enable a ruleset.** Currently running the Emerging Threats Open ruleset.
5. **Services → Intrusion Detection → Download.** Pull down the selected ruleset.
6. **Configure EVE JSON output.** Enable EVE JSON logging, this is what gets shipped to Logstash for ingestion into Elasticsearch.
7. **Apply and start the service.**

---

## Routine tasks

### Updating rulesets

1. Services → Intrusion Detection → Download.
2. Check for and pull the latest ruleset version.
3. Services → Intrusion Detection → Administration → Apply, to reload with the updated rules.
4. Confirm the service restarted cleanly (no errors in the IDS log) and that EVE JSON is still flowing, an interrupted or misconfigured reload can silently stop log shipping.

### Reviewing alerts

Day-to-day alert review happens in Kibana against the ingested EVE JSON (the Suricata SOC dashboard), not in the OPNSense UI directly. Use OPNSense's own Services → Intrusion Detection → Alerts view only for quick spot-checks or when troubleshooting whether Suricata itself is generating alerts at all (as opposed to a downstream shipping issue).

### Troubleshooting a stopped alert flow

If alerts stop appearing in Kibana, check in this order, since it narrows down whether the problem is Suricata itself or the pipeline downstream of it:

1. Is Suricata still running? (Services → Intrusion Detection → Administration)
2. Is EVE JSON still being written locally on OPNSense?
3. Is Logstash still receiving and parsing it? (see [SOC Stack](../../soc-stack.md) for the log flow path)
4. Is the index still being written to in Elasticsearch?

---

## Promoting to IPS (inline blocking) mode — not yet done

This is a deliberate open item, not an oversight, moving from detection-only to inline blocking changes Suricata from a passive observer to something that can actively drop legitimate traffic on a false positive, so it needs a tuning pass first. When ready:

1. Run in IDS mode long enough to have a real baseline of what the ruleset flags in this environment's actual traffic, to catch likely false positives before they can block real traffic.
2. Switch mode to IPS in Services → Intrusion Detection → Administration.
3. Start conservatively, consider running with a subset of rules in blocking mode rather than the full ruleset at once.
4. Monitor closely after the switch for any legitimate traffic being dropped, and be ready to revert to IDS mode quickly if so.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [SOC Stack](../../soc-stack.md)

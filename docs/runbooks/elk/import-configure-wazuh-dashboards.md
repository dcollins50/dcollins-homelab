# Runbook: Import/Configure Wazuh Dashboards (Kibana)

| Field | Value |
| --- | --- |
| Applies to | Kibana, Wazuh Manager, Elasticsearch |
| Category | Dashboards / SOC |
| Author | Daniel Collins |

---

## When to use this

Use this when standing up the Wazuh dashboard set for the first time, adding a dashboard from a newer Wazuh release, or troubleshooting a dashboard that's not rendering against live data. This environment currently runs the official Wazuh dashboard set (Security Events, Malware Detection, Incident Response, PCI-DSS, Vulnerability Management, Docker Listener, OPNSense Firewall) on top of the existing Elastic Stack rather than Wazuh's own bundled Indexer/Dashboard, since this environment's Elasticsearch already serves that role.

---

## Before you start

- **Confirm the Wazuh indexer connector is actually pointed at this environment's Elasticsearch** (by IP, not hostname, see [Configure TLS](configure-tls-elasticsearch-kibana.md) for why), and that alerts are actively flowing before importing dashboards, importing dashboards against an empty index just gets you empty charts, verify data first.
- **Know the Wazuh version's expected dashboard/saved-object export**, dashboard definitions are tied to the Wazuh version, a mismatch between the dashboard export and the running Wazuh Manager version can cause missing fields or broken visualizations.

---

## Steps

### Importing the official dashboard set

1. Obtain the saved-objects export matching the running Wazuh version (from the Wazuh release matching the Manager's installed version, not just whatever's latest).
2. **Kibana → Stack Management → Saved Objects → Import.**
3. Select the export file, import.
4. Resolve any import conflicts (Kibana will flag if an object with the same ID already exists), decide deliberately whether to overwrite or skip each conflict rather than accepting defaults blindly, especially if any dashboards have already been customized here.

### Verifying a dashboard actually works after import

Don't assume "imported without error" means "renders correctly":

1. Open each imported dashboard and confirm panels are actually populated, not blank or showing "no data."
2. Check the data view / index pattern each dashboard's panels reference matches this environment's actual index naming (`wazuh-alerts-*`, etc.), an import from a differently-configured environment can reference an index pattern that doesn't exist here.
3. Spot-check a couple of panels against a known event (e.g. a recent alert you can find directly in Discover) to confirm the dashboard is actually reflecting real data, not just rendering an empty state gracefully.

### Adding a custom panel to an existing Wazuh dashboard

Rather than editing the imported dashboard's underlying saved object directly (risks being overwritten on a future re-import), build the addition as a new Lens visualization (see [Build a Kibana Lens Visualization](build-kibana-lens-visualization.md)) and add it to the dashboard as an additional panel, keeping the original imported panels untouched.

### Troubleshooting a broken/empty dashboard panel

1. Confirm the underlying index actually has data for the field(s) the panel queries, see [Troubleshoot Log Shipping Stopped](troubleshoot-log-shipping-stopped.md) if the whole source looks dark, not just one dashboard.
2. Check the panel's data view matches an index pattern that actually exists, a common cause after a Wazuh version upgrade changes index naming.
3. Check the dashboard's time range, an otherwise-fine panel showing empty is often just scoped outside the range where the relevant data exists.

---

## Common mistakes

- **Importing a dashboard export version-mismatched with the running Wazuh Manager**, causes subtle broken panels rather than an obvious failure.
- **Editing an imported dashboard's saved object directly** instead of adding a new panel, risking the customization being silently lost on the next import/update.
- **Not verifying panels actually show real data post-import**, a dashboard can import cleanly and still be functionally broken if it's pointed at the wrong index pattern.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Build a Kibana Lens Visualization](build-kibana-lens-visualization.md)
- [Troubleshoot Log Shipping Stopped](troubleshoot-log-shipping-stopped.md)
- [Configure TLS](configure-tls-elasticsearch-kibana.md)

# Runbook: Build a Kibana Lens Visualization

| Field | Value |
| --- | --- |
| Applies to | Kibana (Lens) |
| Category | Dashboards / Visualization |
| Author | Daniel Collins |

---

## When to use this

Use this whenever building a new chart/visualization in Kibana Lens for a dashboard, this captures the standing convention already established for how axes are set up, so it doesn't have to be re-derived each time.

---

## Standing convention

**X-axis is the field/dimension. Y-axis is Count of records.**

For a typical "how many of X over time/category" chart, that means:
- **X-axis:** the field being broken out (e.g. `@timestamp` for a time series, or a categorical field like `rule.level`, `source.ip`, etc.)
- **Y-axis:** Count of records (the metric)

This is the default orientation to reach for unless the specific visualization genuinely calls for something else (e.g. a metric that isn't a count, like an average or sum of a numeric field).

---

## Steps

### Creating a new Lens visualization

1. **Kibana → Visualize Library → Create visualization → Lens** (or start directly from a dashboard's "Create panel" if building it in place).
2. Select the relevant data view/index pattern for the source being visualized.
3. **Drag the dimension field to the X-axis** (or the equivalent axis for the chosen chart type).
4. **Set the Y-axis metric to Count of records**, per the standing convention above, unless this specific chart needs a different aggregation.
5. Choose the chart type that fits the data (bar, line, etc., time-series data over a time field usually reads better as a line or area chart, categorical breakdowns usually as a bar chart).
6. Adjust bucketing/interval if it's a time-based X-axis (e.g. auto interval vs. a fixed daily/hourly bucket), pick whatever actually reads clearly at the dashboard's typical time range, not just the default.
7. Name the visualization clearly, tied to what it shows and which log source, consistent with the existing dashboard set's naming.
8. Save, and add to the intended dashboard if not already building in place.

### Adding a filter or breakdown

- Use **Break down by** to split a single chart into multiple series (e.g. by `rule.level`, by host), rather than building separate near-duplicate charts for each value.
- Apply a **filter** at the panel level (not just globally on the dashboard) if this specific visualization should only ever show a subset of the data, regardless of the dashboard's own filter/time range controls.

---

## Common mistakes

- **Swapping the axes** (field on Y, count on X) out of habit from a different tool's convention, breaks consistency with the rest of this environment's dashboards and reads backward.
- **Using Average or Sum when Count was actually the intent**, double-check the Y-axis aggregation type explicitly rather than assuming Lens picked the right default for the field type.
- **Building a near-duplicate chart per value** instead of using Break down by, clutters the dashboard and makes comparison harder than a single multi-series chart would.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Import/Configure Wazuh Dashboards](import-configure-wazuh-dashboards.md)

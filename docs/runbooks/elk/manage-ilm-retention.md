# Runbook: Manage ILM Retention (Elasticsearch)

| Field | Value |
| --- | --- |
| Applies to | Elasticsearch (Index Lifecycle Management) |
| Category | Log Retention |
| Author | Daniel Collins |

---

## When to use this

Use this when setting retention for a new index pattern, adjusting an existing retention window, or diagnosing disk pressure from unbounded index growth. This environment learned this the hard way: no ILM policy existed for months, and unbounded daily indices eventually hit Elasticsearch's `cluster.max_shards_per_node` limit and stopped log ingestion entirely. Retention isn't optional housekeeping here, it's what keeps the cluster functional.

---

## Current policy in this environment

- **General log indices** (Proxmox, OPNSense, Heimdall, etc.): 90-day retention, delete after.
- **`wazuh-alerts`**: 365-day retention, longer window since it's lower-volume, higher-signal incident data compared to raw firewall/traffic logs.
- **`wazuh-archives`**: stays on the 90-day general window.

Any new log source's retention should default to the 90-day general policy unless there's a specific reason (like wazuh-alerts) to extend it.

---

## Steps

### Creating a new ILM policy

1. In Kibana: **Stack Management → Index Lifecycle Policies → Create policy.**
2. Name it clearly, tied to what it governs (e.g. matching this environment's existing naming for the 90-day and 365-day policies).
3. Configure the lifecycle phases needed. For a straightforward retention-only policy (no rollover tiering needed for this environment's scale), the relevant phase is typically just **Delete**, set the minimum age matching the intended retention window (90 or 365 days).
4. Save the policy.

### Applying the policy to an index pattern

ILM policies don't apply themselves, they need to be attached via an index template:

1. **Stack Management → Index Management → Index Templates → Create template** (or edit the existing template if the index pattern already has one).
2. On the template's settings, set `index.lifecycle.name` to the policy created above.
3. Confirm the template's index pattern matches the actual index naming for the log source (e.g. `<source>-logs-*`).
4. Save.

**This only affects indices created after the template is applied.** Indices that already existed before the policy was attached won't retroactively pick it up, see below.

### Applying retention to already-existing indices

For indices that predate the ILM policy (the exact situation that caused the original shard-limit outage here):

1. Confirm which existing indices are NOT yet managed by ILM: **Index Management → Indices**, check the ILM status column.
2. Either wait for the index template fix to apply to future rollovers, or manually set `index.lifecycle.name` on the existing indices if immediate retention enforcement is needed, don't leave old unmanaged indices accumulating indefinitely while only new ones get retention.

### Checking current ILM status

**Index Management → Index Lifecycle Policies**, or **Index Management → Indices** filtered by ILM status, to confirm indices are actually being aged out on schedule, not just that a policy exists somewhere unattached to anything.

---

## Common mistakes

- **Creating the policy but never attaching it to an index template**, this was effectively the original root cause here, a policy that exists but governs nothing doesn't prevent unbounded growth.
- **Assuming a new ILM policy retroactively applies to existing indices.** It doesn't, check and handle pre-existing indices explicitly.
- **Setting every log source to the same retention window without considering signal value**, wazuh-alerts getting a longer window than raw firewall logs was a deliberate call based on data value, not a default, make similarly deliberate calls for new sources rather than copying the same number everywhere by habit.
- **Not periodically checking that ILM is actually deleting on schedule.** A misconfigured policy can silently fail to age out data, verify the count of managed indices isn't just growing forever.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)

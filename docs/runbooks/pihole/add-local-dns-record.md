# Runbook: Add a Local DNS Record (Pi-hole)

| Field | Value |
| --- | --- |
| Applies to | Pi-hole |
| Category | DNS |
| Author | Daniel Collins |

---

## When to use this

Use this whenever an internal service needs a friendly `*.homelab.local` hostname (or any other locally-resolved name) rather than requiring users to remember an IP. This is the record layer that OPNSense's Query Forwarding rule (see [Unbound DNS Forwarding](../opnsense/unbound-dns-forwarding.md)) ultimately points at, that rule forwards the domain, this is what actually answers for a specific hostname within it.

---

## Before you start

- **Know the target IP.** Usually the relevant NPM instance's IP if the service is reverse-proxied (see [Add a Proxy Host](../npm/add-proxy-host.md)), or the service's own IP if reached directly.
- **Confirm the domain is already covered by the Query Forwarding rule** on OPNSense (i.e. `homelab.local` is forwarded to this Pi-hole). If it's a genuinely new domain suffix, that's an OPNSense change, not a Pi-hole one.

---

## Steps

1. **Pi-hole admin → Local DNS → DNS Records** (in newer Pi-hole versions this may be under a slightly different menu path, look for "Local DNS Records" or "Domains").
2. **Add a new record:**
   - Domain: the full hostname (e.g. `service.homelab.local`)
   - IP Address: the target IP
3. **Add.**
4. **Test resolution:**
   ```bash
   dig @<pihole-ip> service.homelab.local
   ```
   Confirm it returns the expected IP, not NXDOMAIN or a fallback/blocked response.

---

## Troubleshooting: record added but not resolving as expected

**Check whether the domain is on a blocklist first.** This is a real, documented Pi-hole behavior: a blocklist match takes precedence over a Local DNS Record, if the exact hostname (or a wildcard covering it) happens to be on an active blocklist, Pi-hole will return a blocked response (`0.0.0.0` or similar) instead of the local record, even though the record exists and looks correct.

1. Check **Query Log** for the specific hostname, if it shows as blocked rather than resolved to the local record's IP, that's the cause.
2. If so, add the exact hostname to the **Whitelist** (Domains → Whitelist), this doesn't disable the blocklist generally, it just carves out this one hostname.
3. Re-test resolution.

This is unlikely for a `*.homelab.local` name specifically (unlikely to collide with a public adlist entry), but worth checking first for anything resembling a real-world domain that might overlap with a blocklist entry.

---

## Common mistakes

- **Assuming a domain not covered by the OPNSense forwarding rule will resolve just because a Local DNS Record exists.** The forwarding rule and the record are two separate layers, both need to be correct.
- **Not checking the blocklist-precedence behavior** when a record that looks correctly configured still doesn't resolve as expected, an easy thing to overlook since the record itself isn't wrong.
- **Forgetting to test with `dig` against Pi-hole directly** first, testing from a client can add ambiguity (client-side caching, wrong resolver in use) that testing directly against Pi-hole avoids.

---

## Related Documentation

- [Unbound DNS Forwarding](../opnsense/unbound-dns-forwarding.md)
- [Add a Proxy Host](../npm/add-proxy-host.md)
- [Network Architecture](../../network.md)

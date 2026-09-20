# Runbook: Unbound DNS Query Forwarding (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense (Unbound DNS Resolver) |
| Category | DNS |
| Author | Daniel Collins |

---

## When to use this

Use this whenever internal hostnames (`*.homelab.local`) need to resolve correctly for VLANs that query OPNSense as their DNS server, or when troubleshooting internal name resolution that isn't behaving as expected. This environment's actual DNS architecture: Pi-hole (on Heimdall) is the primary resolver and ad/DNS filter for all VLANs, and OPNSense's Unbound resolver forwards `homelab.local` queries onward to the internal DNS server. See [network.md](../../network.md) for the full picture.

---

## Before you start

- Know the exact domain or subdomain that needs forwarding, and where it should actually resolve to.
- Confirm whether this is a new forwarding rule, or a fix to an existing one that's pointing at a stale IP (a common cause of "can't reach service X by name" issues after a service moves).

---

## Steps

### Adding or editing a Query Forwarding rule

1. **Services → Unbound DNS → Query Forwarding.**
2. **Add a new domain override** (or edit an existing one):
   - Domain: the domain to forward (e.g. `homelab.local`)
   - Server IP: the internal DNS server that should handle it
3. **Save**, then **Apply Changes**.
4. **Test resolution** from a client on a VLAN that uses OPNSense/Unbound as its resolver:
   ```
   dig @<opnsense-vlan-gateway-ip> some-host.homelab.local
   ```
   Confirm it returns the expected IP, not a timeout or NXDOMAIN.

### Adding a new internal DNS record for a service

Individual internal hostnames (e.g. a new service getting its own `*.homelab.local` name) are managed on the internal DNS server, not in OPNSense directly, OPNSense only forwards the domain, it doesn't hold the records itself. After adding the record on the internal DNS server:

1. Confirm the Query Forwarding rule for the domain is already in place (see above), it usually will be if other `homelab.local` names already resolve correctly.
2. Test resolution of the new hostname specifically, don't assume it works just because other hostnames on the same domain do, a typo in the new record is a different failure mode than a missing forwarding rule.

---

## Troubleshooting

If a `homelab.local` hostname isn't resolving from a particular VLAN:

1. **Confirm the client is actually using OPNSense/Unbound (or Pi-hole, per the resolver chain) as its DNS server**, not some other or hardcoded resolver.
2. **Confirm the firewall permits DNS from that VLAN to OPNSense.** A missing DNS-to-firewall rule on a given VLAN is a firewall issue, not a DNS config issue, see [Add a Firewall Rule](add-firewall-rule.md); this has been the actual root cause of DNS failures here before.
3. **Confirm the Query Forwarding rule exists and points at the right server.**
4. **Confirm the record itself exists and is correct on the internal DNS server**, rule out the forwarding path working correctly but the destination record being wrong or missing.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Add a Firewall Rule](add-firewall-rule.md)

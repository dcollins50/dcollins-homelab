---
name: dns-architecture
type: decision
status: live
---

## Decision
Decided Sep 23, 2026, as part of [[dhcp-dns-rollout]]:
- Unbound on [[opnsense]] becomes the resolver for the stack. [[heimdall]]'s Pi-hole keeps running for domain blocking only.
- Internal names use the `.internal` top-level domain. `homelab.example` is only for internet-facing hosts.
- If Pi-hole is down, DNS fails open, with alerts that it failed.

## Why
- Daniel wants a more mature resolver than Pi-hole as the main one, and one place for internal names. Today Pi-hole and Unbound both hold internal records, which caused a duplicate-answer bug on Sept 15.
- `.local` is the multicast DNS namespace (RFC 6762). Software is expected to resolve it over multicast, not forward it to DNS, so it is unreliable on clients Daniel does not control. This matters more if the system is ever provisioned for another user or org.
- ICANN reserved `.internal` permanently in July 2024 for private-use applications. It resolves through normal DNS.
- Fail open keeps the network working when Pi-hole is down. The risk is silently losing blocking, which is why the alerts are part of the decision.

## Not chosen
- Keeping `.local`.
- `home.arpa` (RFC 8375). Valid, but longer to type.
- Fail closed.
- Keeping Pi-hole as the main resolver.

## Consequences
- Pi-hole sees only OPNsense as a client. Per-client stats and group rules there lose meaning. Per-client visibility would come from Unbound query logging.
- Certificates that name `homelab.local` hosts must be reissued, and configs that reference those names updated. Unbound serves both zones during the transition.
- Authentik is a candidate exception: it is internal-only but uses an `authentik.homelab.example` name whose OIDC URLs are embedded in step-ca and Cloudflare configuration.

## Still open
- Zone apex. Proposed `homelab.internal`, not yet agreed.
- DHCP design details and the phased rollout, in [[dhcp-dns-rollout]].

## Related
- [[dhcp-dns-rollout]]
- [[opnsense]]
- [[heimdall]]
- [[Network]]

# Session Log — September 15, 2026

**Scope:** An overnight session continuing from the Sep 14 decisions (see [Session Log — September 14, 2026](session-log-sept14-authentik-devvlan.md)). Covers the actual Authentik deployment troubleshooting, a real DNS exposure finding, Cloudflare-side hardening for ntfy and the root domain, a Wazuh false-positive triage, and an attack surface inventory.

---

## 1. Authentik Deployment — Troubleshooting Chain

The Authentik LXC itself was already created per the Sep 14 decision (ID 2201, on pve-services, tagged VLAN30/10.0.30.10). Getting it actually reachable and working took several rounds of troubleshooting:

**Certificate:** Issued a custom internal PKI certificate for `authentik.dcollinshomelab.org` through the Intermediate CA. Note: at the time of issuance, the Intermediate CA's actual reachable IP was still 10.0.0.21 (the flat-network address), not the VLAN30 address its documentation implied, another symptom of the CA migration gap flagged on Sep 14.

**504 Gateway Timeout:** After configuring the NPM proxy host, requests to Authentik timed out. Traced through three different network vantage points (pve-services succeeded from the management VLAN, services-host failed from VLAN20) to a missing firewall rule, the Services → DEV pinhole had silently vanished and needed rebuilding.

**502 Bad Gateway:** After fixing the firewall path, a 502 appeared. Traced to NPM's proxy host **Scheme field being set to `https` instead of `http`** for the backend connection, a simple but easy-to-miss misconfiguration.

**Pi-hole split-horizon DNS issue:** AAAA queries for `authentik.dcollinshomelab.org` were returning **both** the local `::1` override and Cloudflare's public IPv6 addresses simultaneously, a merge/caching issue rather than a configuration error. Resolved with a `pihole-FTL` restart.

---

## 2. Critical Finding: Wildcard DNS Exposure

**Discovery:** A wildcard DNS record on `dcollinshomelab.org` was silently making **every** subdomain publicly resolvable, including internal-only hostnames like `authentik.dcollinshomelab.org` that were never meant to have a public presence. This is a real exposure, anything internal named under that domain was effectively discoverable/resolvable from the public internet even though it wasn't meant to be reachable that way.

**Fix:** Removed the wildcard record. Replaced with explicit CNAME records for **only** the root domain and `ntfy`, both pointing at the Cloudflare Tunnel (`<tunnel-uuid>.cfargotunnel.com`). Nothing else gets a public DNS record going forward unless it's deliberately meant to be public.

---

## 3. Wazuh False Positive: `/usr/bin/chsh`

A Wazuh rootcheck alert flagged `/usr/bin/chsh` on pve-services as a potentially trojaned binary. Confirmed as a false positive via `dpkg -V passwd` returning clean output on the correct host, the package's file integrity checked out against its known-good checksums.

---

## 4. Attack Surface Inventory

**Conclusion:** At the time of this session, the actual internet-facing attack surface was **exactly one service: ntfy**, sitting behind Cloudflare's always-on DDoS protection, with no rate limiting, WAF rules, or bot filtering configured yet. Everything else was either not yet deployed publicly, or reachable only through Tailscale, which isn't internet-scannable the way an open port or a proxied hostname is. No SSH bastion existed yet, so there was no direct SSH exposure to the internet at all.

This small, clean surface is what made the Cloudflare hardening below a single-target exercise, everything applied to `ntfy.dcollinshomelab.org` specifically.

---

## 5. Cloudflare Hardening (ntfy)

Four protections stacked against `ntfy.dcollinshomelab.org` in one pass:

1. **Rate limiting** — Security → WAF → Rate limiting rules. Hostname equals `ntfy.dcollinshomelab.org`, 60 requests/minute/IP, Block for 10 minutes on trigger.
2. **WAF managed rules** — Security → WAF → Managed rules. Cloudflare Managed Ruleset (OWASP-style coverage) enabled at default sensitivity.
3. **Bot Fight Mode** — Security → Bots. Enabled (free tier). Challenges traffic lacking normal browser fingerprints.
4. **Geo-blocking** — Security → WAF → Custom rules. `ip.geoip.country ne "US"` → Block, scoped to `ntfy.dcollinshomelab.org`.

**Later extended:** After observing live scanner traffic hitting `/.env` on the root domain, rate limiting and geo-blocking were extended to cover the root domain as well, not just ntfy.

---

## Open items carried forward

- The Intermediate CA's flat-network address (10.0.0.21) used during this session's cert issuance is itself a symptom of the still-pending VLAN30 migration, see [Internal PKI](../../pki.md).
- Verify Bot Fight Mode doesn't interfere with legitimate automated clients (Uptime Kuma's publish calls to ntfy specifically), this was flagged as worth testing but not explicitly confirmed clean in this session.

---

## Related Documentation

- [Session Log — September 14, 2026: Authentik / DEV VLAN Rework](session-log-sept14-authentik-devvlan.md)
- [Incident: ntfy Authentik Outpost Fix — September 15, 2026](../incidents/incident-review-ntfy-authentik-outpost-fix-sept15-26.md) (a separate, later issue on the same day)
- [Internal PKI](../../pki.md)
- [Network Architecture](../../network.md)

# Session Log — September 18, 2026

**Scope:** SSH bastion remote-admin path decision, part of the ongoing SSH Bastion (VLAN51) design work.

---

## Decision: Cloudflare Zero Trust for the Bastion Admin Path

The SSH Bastion's Cloudflare admin tunnel will use **Cloudflare Zero Trust Private Network routing** (Cloudflare One / WARP client, WireGuard-based under the hood) rather than standing up a separate self-hosted WireGuard setup dedicated to that path. Cloudflare Access sits in front, identity-aware per-application auth checks the connection before it ever reaches the bastion's SSH port.

## Decision: Keep All Three Remote-Access Paths in Parallel

Heimdall's WireGuard remote-admin path stays in place alongside this new Cloudflare path and Tailscale, running all three in parallel rather than retiring any of them. Each serves a distinct role rather than being redundant with the others:

- **Cloudflare Zero Trust / WARP** — primary path to the bastion
- **Tailscale** — break-glass path when the bastion itself is down
- **Heimdall's WireGuard** — further fallback when both the bastion and Tailscale are unavailable

This three-path model is what the later SSH bastion MFA design (see [Session Log — September 20, 2026 (AM)](session-log-sept20-bastion-mfa.md)) builds on, including the explicit requirement that none of the three paths have network visibility into each other.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Session Log — September 20, 2026 (AM): SSH Bastion MFA Design](session-log-sept20-bastion-mfa.md)

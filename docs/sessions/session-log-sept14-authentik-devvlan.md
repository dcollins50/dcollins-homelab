# Session Log — September 14, 2026

**Scope:** Authentik deployment and DEV VLAN rework, plus the SSO authorization policy decisions that came with it. Separate from the same-day DNS/firewall incident, which is documented in [incident-review-sept14.md](../incidents/incident-review-sept14.md).

---

## 1. DEV VLAN Repurposed as Trust Infrastructure

**Decision:** The DEV VLAN is repurposed as a dedicated trust infrastructure VLAN, housing both internal CAs and Authentik, kept separate from general application hosting on the Services VLAN. General app hosting stays on Services; anything that other services need to *trust* (CAs, IdP) moves to its own segment.

**Discovery during this work:** The switch's actual VLAN ID for "Dev" is **30** (10.0.30.0/24), not 6 as OPNSense's interface naming (`vlan06`) suggested. Confirmed switch VLAN table:

| VLAN ID | Name |
|---|---|
| 1 | Default |
| 10 | SOC |
| 20 | Services |
| 30 | Dev (now Trust Infrastructure) |
| 40 | Security1 |
| 41 | Security2 |
| 50 | DMZ |
| 60 | Storage |

**Discovery, flagged as an open item:** The Root CA (VM 500) and Intermediate CA (VM 501) were never actually migrated to VLAN 30, despite documentation assuming they had been. Both were still untagged on the flat 10.0.0.x scheme, riding a full trunk port with no VLAN tag set on `net0`. Flagged to fix when DEV/Trust Infrastructure firewall rules get properly rescoped, not yet resolved as of this session.

## 2. Authentik Deployed

Authentik deployed as an LXC (ID 2201, on pve-services, correctly tagged VLAN 30) for self-hosted IdP/SSO, sized for the actual light-scale use case (4-5 users), not over-provisioned for a scale this environment doesn't need.

## 3. Authentik SSO Authorization Policy

**Decision:** Default Authentik authorization flow for internal applications uses **implicit consent**, not explicit. Rationale: all users and applications in this environment are self-owned and trusted, an explicit per-app consent prompt adds friction without adding real security value here.

**In progress:** ntfy being put behind Authentik forward-auth (proxy outpost), scoped so the browser web UI requires login, while Uptime Kuma's automated publish calls bypass the outpost entirely and use ntfy's own token auth directly, automation shouldn't be forced through an interactive auth flow it can't complete.

---

## Open items carried forward

- Root CA / Intermediate CA still not actually migrated to VLAN 30 (Trust Infrastructure), despite that being the intent, see [Internal PKI](../../pki.md) for current documented state and the step-ca migration work that later builds on this.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Network Architecture](../../network.md)
- [Add an Application (Authentik)](../runbooks/authentik/authentik-add-application.md)

---
name: sop-ssh-bastion-build
type: sop
status: in-progress
---

# SOP: SSH Bastion Build (VLAN51)

**Status:** In progress. VLAN51, firewall scoping, and the step-ca/Authentik trust pieces are done. The bastion LXC (2202, 10.0.51.10) is built as of Sep 23, 2026. State as of the Sep 21, 2026 session, the last one worked (confirmed by Daniel on Sep 24): WARP device enrollment is blocked by a browser-side `ERR_QUIC_PROTOCOL_ERROR` on the final authorize redirect, sshd cert trust is not configured, the temporary workstation to bastion port 22 rule is not recorded as removed, and the SSH cutover has not happened.
**Objective:** A single, identity-verified SSH entry point for every internal node, replacing static SSH keys with short-lived certificates and requiring two independent MFA layers.

---

## Why this is an SOP and not a runbook

The bastion touches five independently-built systems that all have to agree before it works: [[VLAN51]], [[opnsense]] firewall scoping, [[authentik]] as IdP, [[pve-int-stepca]] as the SSH cert issuer, and Cloudflare Zero Trust/WARP as the tunnel. None of those systems' own runbooks cover how they combine for this specific purpose.

## Architecture (as designed)

```
Admin workstation
    │  WARP client, enrolled via Authentik (MFA layer 1)
    ▼
Cloudflare Zero Trust (Access + WARP Connector, outbound-only from bastion)
    │
    ▼
SSH Bastion (10.0.51.10, VLAN51) — runs the WARP Connector itself
    │  short-lived SSH cert, issued by step-ca via its own OIDC login
    │  against Authentik (MFA layer 2)
    ▼
Internal nodes (bastion IP becomes the only permitted SSH source, once live)
```

Auth requires **two separate MFA layers**, not one MFA prompt reused twice: WARP client enrollment is one check, and step-ca issuing the short-lived SSH cert via its own independent OIDC login against Authentik is the second.

## What's already built

1. **VLAN51 (DMZBastion) created** — isolates the bastion from the rest of [[VLAN50]] DMZ. Added to switch trunk ports 2 (pve-services) and 5 (OPNSense LAN). OPNSense interface `vlan09`, parent `re0`, tag 51, static gateway 10.0.51.1/24. Deliberately fully static for now — DHCP deferred to a future project.
2. **Firewall rules scoped to a `bastion` host alias (10.0.51.10):**
   - DNS to the OPNSense resolver
   - WARP tunnel to Cloudflare via `cf_warp_ingress`/`cf_warp_ports` aliases
   - WARP client API and DoH via `cf_warp_api`/`cf_warp_doh` aliases
   - Authentik OIDC via `internal_npm`
   - step-ca via `pve_int_stepca`
3. **step-ca's Authentik OIDC provisioner registered** (client ID, config endpoint, `enableSSHCA: true`), confirmed via `step ca provisioner list`.
4. **Decided:** bastion will be a native-install LXC (not Docker-in-LXC), to avoid unofficial/fragile WARP-in-Docker tooling. Will run on [[pve-services]] (ruled out pve-env1/pve-env2 as near RAM capacity, pve-gateway as security-lab-only).
5. **Decided:** the bastion runs the Cloudflare WARP Connector itself (outbound-only tunnel to Cloudflare), rather than routing admin traffic through a separate connector host.
6. **Break-glass paths locked in:** Heimdall's WireGuard is the secondary path if the bastion is down, and Tailscale is the tertiary break-glass path if both are down (order set Sep 24, 2026, see [[remote-access-path-order]]; the Sep 18 to Sep 23 wording had them the other way round). The bastion will have no route to Tailscale, and the tailnet reaching the bastion is acceptable (relaxed Sep 23, see [[tailscale-access-model]]). WireGuard has no route to or from the bastion — the paths stay separate on the bastion's outbound side.

## Still open

- **Next step (from the Sep 21 session):** test the exact failing `/application/o/authorize/` URL with `curl`, which does not use QUIC, to confirm whether the failure is QUIC-specific.
- **Configure sshd** on the bastion to trust step-ca's SSH user CA (`TrustedUserCAKeys`). Not started.
- **Remove the temporary bootstrap rule** (`workstation` to `bastion`, port 22 on DMZBastion) once enrollment works.
- **The bastion LXC (2202) is now built** — VLAN51 and its firewall rules exist, and the host at 10.0.51.10 is up. The remaining pieces below are not done (state as of the Sep 21 session).
- **WireGuard fallback vs the SSH cutover:** once the bastion IP is the only permitted SSH source to internal VLANs, SSH from a WireGuard peer would also be blocked unless carved out. Raised Sep 23. It is not yet decided what the WireGuard path should be able to reach.
- **apt package update rule** — leaning toward a Debian repo host alias, not yet finalized.
- **Bastion NTP source** — undecided whether it syncs against OPNSense's internal NTP server or reaches the internet directly.
- **Once the bastion is live:** direct workstation-to-internal SSH will be blocked, and the bastion IP becomes the sole permitted SSH source to internal VLANs. That's a hard cutover step worth its own checklist when the time comes.

## Related Documentation

- [[pve-bastion]]
- [[VLAN51]]
- [[VLAN50]]
- [[pve-services]]
- [[authentik]]
- [[pve-int-stepca]]
- [[opnsense-isolated-vlan|Build a New Isolated VLAN Segment]]
- [[cloudflare-warp-zero-trust-enrollment|Cloudflare WARP Enrollment]]
- [[authentik-oidc-external-service|Add a Dedicated OIDC Provider for an External Service]]

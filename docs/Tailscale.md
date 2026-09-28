---
name: Tailscale
type: moc
status: live
---

Tailscale setup and its role in remote administration. Host-specific detail lives in the linked Hosts/ notes. Written Sep 23, 2026 from that day's troubleshooting sessions and a screenshot of the Tailscale admin console; anything not confirmed is marked as such.

## Role
Tailscale is the break-glass remote-admin path. It is one of three parallel paths:

| Path | Purpose | Notes |
|------|---------|-------|
| Cloudflare WARP into the SSH bastion (bastion built on VLAN51, WARP enrollment blocked, cutover not started) | Primary | Authentik as Zero Trust IdP, plus a step-ca issued short-lived SSH cert as a second MFA event |
| WireGuard on [[heimdall]] | Secondary, when the bastion is unavailable | In progress. Endpoint config is stale and not yet properly configured |
| Tailscale | Tertiary, break-glass when both the bastion and WireGuard are unavailable | This note |

Order set Sep 24, 2026, see [[remote-access-path-order]].

## Design rule
The bastion never joins the tailnet and never gets a tailnet address, and it must not be able to reach the tailnet. The tailnet reaching the bastion is acceptable (relaxed Sep 23, 2026 from the earlier rule that neither side could see the other). Enforced outside Tailscale, on [[opnsense]] and by keeping Tailscale off the bastion, since a Tailscale policy cannot restrict a device that is not enrolled. See [[tailscale-access-model]]. The bastion is built (confirmed Sep 23). Tailscale is not installed on the bastion itself, but is installed on its host node [[pve-services]].

## Enrolled machines (admin console, Sep 23, 2026)
Eight machines, all showing Connected, all under one account.

| Machine | Tailscale IP | Client version | Notes |
|---------|--------------|----------------|-------|
| <host-redacted> | <tailscale-ip> | 1.102.2 at screenshot | Windows 11 workstation, updated since (new version not verified) |
| [[heimdall]] | <tailscale-ip> | 1.102.4 | Tailscale SSH server enabled, intentional |
| jetson | <tailscale-ip> | 1.102.4 | See [[jetson-orin-nano]] |
| [[pve-env1]] | <tailscale-ip> | 1.102.4 | |
| [[pve-env2]] | <tailscale-ip> | 1.102.4 | |
| [[pve-gateway]] | <tailscale-ip> | 1.102.4 | |
| [[pve-services]] | <tailscale-ip> | 1.102.4 | |
| [[services-host]] | <tailscale-ip> | 1.102.4 | Was logged out earlier the same day, see below |

All four Proxmox nodes are on the tailnet. [[opnsense]] is not.

## What went wrong on services-host (earlier Sep 23)
- `tailscale status` reported logged out with `fetch control key ... tls: unrecognized name`.
- Cause identified: Pi-hole on [[heimdall]] was sinkholing Tailscale's control domains, so the connection landed on something that rejected the hostname instead of Tailscale's servers.
- A Pi-hole whitelist fix was applied. Pi-hole is the central resolver, so it applies network-wide.
- `tailscale up` still hung after that fix. It now shows Connected in the admin console. What finally cleared the hang was not recorded.

## Diagnostic order for next time
1. Resolve the control domain and confirm it returns a public IP, not a 10.x or 192.168.x address.
2. Test the HTTPS connection directly with curl.
3. Check `tailscaled` service state.
4. Only then look at OPNsense rules and Suricata.

## Policy
A restrictive policy was applied Sep 23, 2026, replacing the default allow-all. Workstation only as a source, SSH and Proxmox UI to the PVE nodes, SSH only to services-host, jetson and heimdall. Full policy and verification in [[tailscale-tailnet-policy|Apply the Tailnet Policy]]. Rules and reasoning in [[tailscale-access-model]].

## Decisions
- Sep 23, 2026: Tailscale stays as the break-glass admin path. Decommission option closed.

## Not yet documented
- The applied policy only lets the workstation act as a source. A phone or laptop needs a host alias and a grant when it enrolls. See [[tailscale-tailnet-policy|Apply the Tailnet Policy]].
- Key expiry: reported done Sep 23, 2026 (not verified in the console). Which nodes it was disabled on was not specified.
- What the Tailscale SSH server on heimdall is scoped to (intentional, confirmed Sep 23)
- Route advertisement, checked Sep 23: no node has an approved subnet route or exit node (seven nodes checked from pve-services, pve-services checked from pve-env1). Only approved routes show in that check. Re-run it if a node is ever re-enrolled or a route is approved.

## Related
- [[Infrastructure]]
- [[pve-bastion]]
- [[tailscale-tailnet-policy|Apply the Tailnet Policy]]
- [[Network]]
- [[heimdall]]
- [[opnsense]]

---
name: session-2026-09-23-tailscale-policy
type: session
date: 2026-09-23
---

# Session Log: September 23, 2026 (evening): Tailscale Documentation and Tailnet Policy

**Scope:** Filled in the empty `Tailscale.md`, decided the tailnet access model, wrote and applied a restrictive tailnet policy, and brought the bastion notes up to date.

## What was done
- Confirmed the filesystem connector reaches the vault. Found `Tailscale.md` was a 0 byte stub and wrote it as a MOC note from that day's chats and an admin console screenshot.
- Recorded the access model in [[tailscale-access-model]]. Tailscale stays as the break-glass path. The bastion never joins the tailnet and must not reach it. The tailnet reaching the bastion is acceptable. Heimdall's WireGuard stays blocked from the bastion in both directions.
- Wrote and applied the tailnet policy, replacing the default allow-all. Verified from the workstation and from services-host. Details in [[tailscale-tailnet-policy|Apply the Tailnet Policy]].
- Checked route advertisement. No node has an approved subnet route or exit node.
- Created [[pve-bastion]] (LXC 2202, 10.0.51.10, VLAN51). Updated the bastion SOP and [[VLAN51]] to match.
- Key expiry reported done by Daniel. Which nodes was not specified.

## Corrections found in the vault, not yet fixed
- pve-env2.md links `[[soc-stack]]`, but the file is `soc-stack-vm.md`.
- Chat history mentions a Heimdall WireGuard endpoint of 192.168.1.2. Daniel says that is stale and not yet configured. The vault's 192.168.100.1 was left unchanged.

## Open items
- Whether key expiry is disabled on the workstation as well as the servers.
- Optional OPNsense rule blocking the bastion to <tailscale-ip>/10, as a backstop.
- What the WireGuard fallback should reach after the SSH cutover. It is a note in the bastion SOP.
- Bastion SOP items not yet confirmed done: WARP Connector, sshd cert trust, SSH cutover.

## Related
- [[Tailscale]]
- [[tailscale-access-model]]
- [[pve-bastion]]
- [SSH Bastion Build](../SOPs/sop-ssh-bastion-build.md)

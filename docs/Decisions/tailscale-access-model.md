---
name: tailscale-access-model
type: decision
status: live
---

## Decision
Tailscale stays as the break-glass admin path for the hosts themselves. It is the third path, after the bastion and WireGuard (order set Sep 24, 2026, see [[remote-access-path-order]]). It is not a general-purpose network. Access on the tailnet is restricted by a written policy, not left on the default allow-all.

## Rules agreed (Sep 23, 2026)
- Only the Windows workstation is an admin client for now. Phone and laptop get added when they enroll.
- The bastion never joins the tailnet and is never assigned a tailnet address. The bastion must not be able to reach the tailnet.
- The tailnet being able to reach the bastion is acceptable. This relaxes the earlier rule from the Sep 18 to Sep 20 bastion design sessions, which said neither side could see the other.
- Heimdall's WireGuard has no route to or from the bastion, in either direction. Decided Sep 23, 2026: nothing good comes from those paths knowing about each other.
- Tailscale is installed on the bastion's host node ([[pve-services]]), not on the bastion LXC.
- The jetson is reachable over SSH. The workstation must be able to diagnose connection issues to the nodes (exact tooling not specified).
- Heimdall runs a Tailscale SSH server on purpose.

## Why
The tailnet is primarily for the hosts themselves, as a break-glass path when the bastion is down. The bastion is the hardened entry point with two MFA layers (see the bastion SOP), so it must not gain a route into the tailnet. The tailnet reaching the bastion was explicitly judged acceptable.

## How the bastion rule is enforced
A Tailscale policy cannot restrict a device that is not on the tailnet, so enforcement lives elsewhere:
- No Tailscale client on the bastion LXC.
- No subnet router advertising the bastion's subnet. Do not run route advertisement on [[pve-services]].
- The firewall rules on [[opnsense]] scoped to the bastion alias stay deny by default. Proposed, not yet confirmed: an explicit block to the Tailscale address range as a backstop.

## Ruled out
- Decommissioning Tailscale. Decided against on Sep 23.
- Extending the relaxed bastion rule to Heimdall's WireGuard. Decided against on Sep 23. It would also have given a network path to the bastion's sshd that skips the WARP MFA layer.
- Leaving the default allow-all policy. Every enrolled machine could reach every other one on every port.

## Related
- [[Tailscale]]
- [[heimdall]]
- [[opnsense]]
- [[sop-ssh-bastion-build|SSH Bastion Build]]

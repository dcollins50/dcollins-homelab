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

## Exception (Sep 28, 2026)
One scoped exception to "admin only": [[dev-git]] is shared to one outside user through Tailscale machine sharing, limited to Gitea's web and SSH ports, with a policy test proving the box can't reach any other tailnet machine. See [[gitea-collab-access]].

## Second exception (Sep 29, 2026, live Sep 30, 2026)
[[nimbus]] is a second scoped exception on the same terms: one non-admin machine shared to the same outside user, limited by policy grants to its served ports (`tcp:80` and `tcp:443`; the container firewall accepts only 443), with a policy test that it cannot reach any other machine. See [[cloud-files-design]].

## Third exception (Oct 23, 2026)
`[admin-user]-worktop` (laptop, 100.x.x.x) is a source for [[nimbus]] on `tcp:443` only. No SSH, no Proxmox, no other hosts, so the "only the workstation is an admin client" rule still holds. The test block asserts it cannot reach nimbus:22, nimbus:80, pve-env1:22, pve-env1:8006 or workstation:22. Remote admin from a laptop still goes through the bastion path, not Tailscale. Open: whether [admin-user]-worktop is employer-managed, which would change the risk of a logged-in session to files shared with the collaborator.

## Further exceptions found in the live policy (recorded Nov 7, 2026)
These were applied at an unrecorded date between Oct 23 and Nov 7, 2026, and were not written up at the time. Copied from the live policy, see [[tailscale-tailnet-policy]].
- **achilles:** a machine shared in from the collaborator's tailnet. The workstation and [admin-user]-worktop reach its web page on `tcp:80` only, with tests that `achilles:22` is denied.
- **Asscord staging VPS:** a Hostinger server tagged `tag:asscord-staging` (tag owner `autogroup:admin`). The workstation and [[pve-ansible]] reach it on `tcp:22` only. Nothing grants the VPS any access. No test yet proves that.
- **pve-ansible to dev-git:** `tcp:2222` only (Gitea SSH), to clone `[admin-user]/asscord` with a read only deploy key. Tested: `dev-git:22` and `dev-git:3000` denied.

## Stance as of Nov 7, 2026
Daniel says the tailnet now gets daily use for fringe machines and he is not worried about it. That moves away from the break-glass-only wording in the Decision section above. The rule that held through every exception: each new source gets only named destinations and ports, with tests.

Planned: [[homelab-update-automation]] moves the Asscord deploys from pve-ansible to `ansible-lab` (LXC 1200). Agreed Nov 7, 2026: ansible-lab gets a one-way exception, reaching the Asscord VPS on SSH (and dev-git on 2222 for the repo) and nothing reaching it, with tests.

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

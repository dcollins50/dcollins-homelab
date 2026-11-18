---
name: gitea-collab-access
type: decision
status: live
---

## Decision
Decided Sep 28, 2026. To collaborate with one outside person (a friend who does not want to use GitHub), a separate Gitea instance is built just for collaboration, [[dev-git]], instead of opening the existing Gitea on [[services-host]]. He reaches it through Tailscale machine sharing: only that one machine is shared into his own tailnet, and only on Gitea's web and SSH ports.

- Runs as its own Debian 13 LXC on [[pve-env1]], in the DMZ ([[VLAN50]]), Gitea installed natively (no Docker).
- Only one outside user for now. If more people need access later, this gets rebuilt properly with [[authentik]] issuing identities and group memberships that grant Gitea access, instead of one Tailscale share per person.
- Placement on pve-env1 checked against the Proxmox node summary on Sep 28, 2026: pve-env1 at 56% memory, pve-env2 at 82%, pve-services at 40%, pve-gateway at 19%.

## Why
- The risk being managed is not the friend, it's his device or his Tailscale account being taken over. Whoever holds either gets exactly the access granted. On a separate box, that's one project. Shared services-host would have put that access on the same machine as Vaultwarden, held back only by a port rule.
- The DMZ already blocks all traffic into internal VLANs and allows outbound internet, which is all Tailscale needs. So no new [[opnsense]] rules are needed, and a compromised box has nowhere to go.
- Tailscale shared machines can't start connections back into the recipient's tailnet. The existing tailnet policy only allows the workstation as a source, so dev-git can't reach any other tailnet machine either.
- Tailscale was picked over a public Cloudflare hostname because it needs no public login page and no extra Cloudflare client for git over SSH.

## Exception to the tailnet access model
[[tailscale-access-model]] says the tailnet is admin-only, a break-glass path, not a general-purpose network. This decision is a deliberate, scoped exception: one non-admin machine, shared to one outside user, limited by policy grants to two ports, with policy tests that fail the save if anything reaches further.

## Ruled out
- **Sharing services-host over Tailscale.** It puts the outside user on the same machine as Vaultwarden, Portainer and the internal Gitea.
- **Publishing the existing Gitea through the Cloudflare web tunnel with Cloudflare Access.** Needs a new DMZ-to-VLAN20 allow rule, breaks the tunnel SOP's rule that nothing internet-facing shares infrastructure with the git server, and git over SSH through Cloudflare needs extra client software.
- **A WireGuard peer on [[heimdall]].** That path is stale, and it's an admin path that would put him on the network rather than in front of one service.
- **pve-env2** (82% memory, reserved for the SOC stack) and **pve-services** (holds the CAs, ruled out for outside-facing services in [[sop-cloudflare-tunnel-implementation]]).

## Related
- [[sop-gitea-collab-build]]
- [[dev-git]]
- [[tailscale-access-model]]
- [[tailscale-tailnet-policy|Apply the Tailnet Policy]]
- [[gitea-enable-mfa]]

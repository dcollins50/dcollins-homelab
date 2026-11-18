---
name: stepca-remote-access-via-warp
type: decision
status: live
---

## Decision
Daniel decided on Oct 17, 2026 that a device away from home (the laptop he is about to buy) gets its bastion SSH certificate by reaching [[pve-int-stepca]] through the Cloudflare One Client, as an enrolled device. step-ca is never published to the internet. No public hostname, no public route.

Tailscale is not used as the path for this or for any other access. It stays break glass only. The current Tailscale share used with the collaborator for dev-git was a quick way to get something running and is not a pattern to build on. See [[tailscale-access-model]].

## Why
- A fresh certificate needs the laptop to talk to step-ca at the moment it asks. With step-ca reachable only on the LAN, a remote device could not get one at all.
- The Authentik half already works from outside. The provisioner's configuration endpoint is `https://authentik.homelab.example/application/o/step-ca/.well-known/openid-configuration`, published through [[pve-authtunnel]] ([[session-2026-09-21-stepca-and-bastion]]). Only step-ca itself (`https://stepca.homelab.local:9000`, 10.0.30.x) is missing a path.
- Reachable only by enrolled devices keeps step-ca off the internet while keeping the short-lived certificate design.

## Ruled out
- Public exposure of step-ca, including through a Cloudflare Tunnel public hostname behind Access.
- Tailscale or Heimdall WireGuard as the daily path. Both stay break glass.
- A long-lived certificate issued at home before a trip. It breaks the short-lived design.

## Connector (chosen Oct 17, 2026)
A separate small connector LXC (`pve-stepca-tunnel`, VMID 2101, on [[pve-services]]) running its own cloudflared tunnel, with a private route for 10.0.30.x/32 only and no public hostname. It forwards only to step-ca on port 9000.

**Placement (changed the same day):** first planned on [[VLAN30]], then moved to a new shared VLAN for tunnel containers only. Reason: hosts in the same subnet talk to each other directly and never pass through OPNsense, so a connector on VLAN30 could reach step-ca, Authentik and both CAs with no firewall in between. On its own VLAN, every path from a tunnel to its target crosses OPNsense, where it is allowed by a scoped rule and logged. Tunnel containers on the shared VLAN can still reach each other directly, which was accepted. One VLAN per tunnel was ruled out as too much overhead per tunnel. The Proxmox firewall per container was ruled out as the main control because its logs stay in Proxmox.

Ruled out for the connector:
- The bastion as a Mesh node advertising a route: it would forward into VLAN30, open its nftables forward chain and need an OPNsense return route, which undoes its isolation on [[VLAN51]].
- cloudflared on [[pve-int-stepca]] itself: a compromise of the tunnel would land on the CA host.
- Adding the route to [[pve-authtunnel]]: that tunnel fronts the public identity provider, a different trust boundary.

**Values (Oct 17, 2026):** VLAN 52, named `DMZTunnels`, subnet 10.0.52.0/24, gateway 10.0.52.x, tagged on switch trunk ports 2 (pve-services) and 5 (OPNsense LAN). `pve-stepca-tunnel` at 10.0.52.x. [[pve-authtunnel]] moves to 10.0.52.x later, as a separate step after the step-ca path is proven, so the two are never debugged at once.

## Not decided yet
- How the laptop resolves `stepca.homelab.local`.
- Split tunnel change: the default Exclude list keeps RFC 1918 space local, so 10.0.30.x has to be carved out of the 10.0.0.0/8 exclude for traffic to go through the client.
- The Gateway Allow rule for 10.0.30.x port 9000, which must sit above "Default deny for private traffic" ([[bastion-host-firewall]]).

## Related
- [[bastion-ssh-certificate-login]]
- [[pve-bastion]]
- [[pve-int-stepca]]
- [[cloudflare-warp-windows-workstation-mesh-access]]

---
name: pve-authtunnel
type: lxc
vmid: 2100
node: pve-services
ip: 10.0.50.70
vlan: VLAN50
status: live
role: Dedicated Cloudflare Tunnel that publishes Authentik for Cloudflare Access
---

Unprivileged LXC built Sep 21, 2026. Runs `cloudflared` as a systemd service for a second, separate Cloudflare Tunnel (`auth-tunnel`), kept apart from the public web tunnel on [[npm-dmz]] so the path that exposes the identity provider does not share a host with public content. Daniel confirmed on Sep 24, 2026 that it still exists.

**Why it exists:** Cloudflare Access does its OIDC token exchange from Cloudflare's own backend, so [[authentik]] had to be reachable from the public internet. It had only ever resolved internally.

**Route:** `authentik.homelab.example` goes to internal NPM on [[services-host]] (10.0.20.30, port 443), not to Authentik directly. Authentik's own listener on 10.0.30.10:9443 uses a self-signed certificate, which made the first attempt return Bad Gateway. NPM terminates TLS with a certificate from the internal CA.

**CA trust:** the homelab root certificate was copied onto this LXC through the Proxmox host (`pct push 2100`, see [[proxmox-cross-vlan-file-transfer]]) because DMZ has no route to the trust infrastructure VLAN. The route's CA Pool points at `/etc/ssl/certs/homelab-root-ca.crt` and its Origin Server Name is `authentik.homelab.example`. TLS verification is left on.

**Firewall (VLAN50 interface):** DNS and NTP to the VLAN gateway 10.0.50.1, outbound to the Cloudflare edge range (`cf_edge_ipv4`), and a rule to the `internal_npm` alias on 443. It is not part of the bastion's isolation and has no route to [[VLAN51]].

Verified from cellular data on Sep 21: `https://authentik.homelab.example/.well-known/openid-configuration` returns clean JSON.

Details of the build are in [[session-2026-09-21-stepca-and-bastion]]. The general publishing procedure is [[cloudflare-tunnel-publish]].

## Related
- [[pve-services]]
- [[VLAN50]]
- [[authentik]]
- [[npm-dmz]]
- [[services-host]]

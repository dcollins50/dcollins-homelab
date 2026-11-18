---
name: authentik
type: lxc
vmid: 2201
node: pve-services
ip: 10.0.30.x
vlan: VLAN30
status: live
role: Self-hosted IdP/SSO
---

Deployed Sep 14, 2026 for self-hosted IdP/SSO, light-scale use case (4-5 users). Correctly tagged VLAN30 from deployment.

**Authorization flow:** implicit consent (not explicit) as the default for all internal applications, since all users and apps are self-owned and trusted.

**MFA:** TOTP/authenticator set up.

**ntfy integration:** ntfy sits behind Authentik forward-auth (proxy outpost) — the browser web UI requires login, but Uptime Kuma's automated publish calls bypass the outpost and use ntfy's own token auth directly.

**OIDC provisioners registered:**
- [[pve-int-stepca]] — for SSH bastion cert issuance (enableSSHCA true)

**Providers and applications (confirmed in vault and UI, Oct 17, 2026):**
- `step-ca`: OAuth2/OIDC provider, own application `step-ca`. Redirect URIs: `https://stepca.homelab.local:9000/` (Strict) and `http://127\.0\.0\.1:\d+/?` (Regex, Authorization) for the step CLI loopback ([[session-2026-10-03-stepca-ssh-ca]]). Authorization flow: `step-ca-authorization-mfa` since Oct 17, 2026 (Authorization designation, Denied action MESSAGE, one stage bound at order 10: `step-ca-mfa-every-time`, Authenticator Validation, TOTP only, Not configured action Deny, Last validation threshold `seconds=0`). Every certificate request asks for TOTP, even with an active Authentik session; tested Oct 17. Before that it was `default-provider-authorization-implicit-consent` (Authorize Application), which is the rollback value. This is the only provider not on implicit consent.
- `cloudflare-access`: separate provider and application, used as the Cloudflare Access / WARP enrollment IdP ([[session-2026-09-21-stepca-and-bastion]]).

No Windows AD domain for identity — the planned Active Directory lab (on [[pve-services]]) is purely an attack-range target, unrelated to this production IdP.

## Update Oct 31, 2026: resolver moved to Unbound
DNS now points at 10.0.30.x (Unbound on the VLAN30 gateway). Before: 192.168.100.x (Pi-hole). Set in the Proxmox container config with `pct set 2201 --nameserver 10.0.30.x` on [[pve-services]]; search domain `homelab.local` unchanged. A backup of the old container config is at `~/2201.conf.bak-2026-10-31` on pve-services. Admin login with TOTP tested working after the change. Rollback: `sudo pct set 2201 --nameserver 192.168.100.x`, then fix `/etc/resolv.conf` inside the container. Part of [[dns-phase1-consolidation]]. Authentik runs as three Docker containers inside this LXC (`authentik-server-1`, `authentik-worker-1`, `authentik-postgresql-1`, all up 13 days and healthy on Oct 31, 2026). Checked at 09:43: `docker exec authentik-server-1 getent hosts stepca.homelab.internal` returned nothing, so the containers still use the old Pi-hole path (Pi-hole cannot see `.internal` names). That is harmless because Pi-hole still answers. They move to Unbound at their next restart. No restart was done for this.

## Runbooks
- Runbooks/authentik/ (add application, backup/restore, configure outpost, enforce MFA, manage permissions, manage users, diagnose flow failures, OIDC for external service)

## Related
- [[pve-services]]
- [[pve-int-stepca]]
- [[VLAN30]]

---
name: authentik
type: lxc
vmid: 2201
node: pve-services
ip: 10.0.30.10
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

No Windows AD domain for identity — the planned Active Directory lab (on [[pve-services]]) is purely an attack-range target, unrelated to this production IdP.

## Runbooks
- Runbooks/authentik/ (add application, backup/restore, configure outpost, enforce MFA, manage permissions, manage users, diagnose flow failures, OIDC for external service)

## Related
- [[pve-services]]
- [[pve-int-stepca]]
- [[VLAN30]]

---
name: VLAN51
vlan_id: 51
subnet: 10.0.51.0/24
status: live
role: DMZ2 / SSH Bastion
---

Created Sep 21, 2026 (named DMZBastion) to isolate the bastion from the rest of [[VLAN50]]. Added to switch trunk ports 2 (pve-services) and 5 (OPNSense LAN). OPNSense interface: vlan09, parent re0, tag 51, static gateway 10.0.51.1/24. Deliberately fully static for now — DHCP deferred to a future project.

**Status (Sep 24, 2026):** [[pve-bastion]] (LXC 2202) is built. As left in the Sep 21 session, WARP enrollment is blocked by a QUIC error on the authorize redirect, sshd cert trust is not configured, and no SSH cutover has happened, so the policy below is still the plan. The temporary `workstation` to `bastion` port 22 rule is not recorded as removed.

**Firewall policy:** designed as the single SSH entry point for all internal nodes. Once live, the bastion IP will be the only permitted SSH source to internal VLANs; direct workstation-to-internal SSH will be blocked at that point.

Bastion host alias (10.0.51.10) rules already scoped: DNS to OPNSense resolver, WARP tunnel to Cloudflare (cf_warp_ingress/cf_warp_ports), WARP client API/DoH (cf_warp_api/cf_warp_doh), Authentik OIDC via internal_npm, step-ca via pve_int_stepca. Still open: apt package update rule (leaning Debian repo host alias), and whether bastion NTP syncs against OPNSense's internal NTP vs. the internet.

Auth design: two MFA layers — WARP client enrollment via [[authentik]] as IdP, plus [[pve-int-stepca]] issuing short-lived SSH certs via its own OIDC login against Authentik, replacing a static SSH key. Will run the Cloudflare WARP Connector itself (outbound-only tunnel), not a separate connector host.

The bastion will have no route to [[Tailscale]] (break-glass), and the tailnet reaching the bastion is acceptable (relaxed Sep 23, see [[tailscale-access-model]]). Heimdall's WireGuard has no route to or from the bastion. Decided Sep 23 to keep it that way.

## Hosts
| Host | IP |
|------|----|
| [[pve-bastion]] | 10.0.51.10 |

## SOPs
- [[sop-ssh-bastion-build|SSH Bastion Build]]

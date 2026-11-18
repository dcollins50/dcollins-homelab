---
name: pve-stepca-tunnel
type: lxc
vmid: 2101
node: pve-services
ip: 10.0.52.x
vlan: VLAN52
status: building
role: Cloudflare Tunnel connector giving enrolled devices a private route to step-ca only
---

Unprivileged Debian 13 LXC, nesting on, 1 core, 512 MB RAM, 512 MB swap, 4 GB on `local-lvm`, DNS 10.0.52.x, search domain `homelab.internal` (set Oct 17, 2026). IPv6 disabled by `/etc/sysctl.d/99-disable-ipv6.conf`. apt sources on https. Debian updates go through the DMZTunnels rule `pve_stepca_tunnel` to `debian_repos` (Fastly ranges) TCP 443; `cloudflared` updates through the `cf_edge_ipv4` 443 rule. No temporary rule needed for `apt upgrade` (since Oct 17, 2026). Created Oct 17, 2026. Managed with `pct enter 2101` from [[pve-services]].

Runs its own cloudflared tunnel (`stepca-tunnel`, created Oct 17, 2026, dashboard-managed, `cloudflared` 2026.10.1 as the enabled `cloudflared.service`) with a private route for 10.0.30.x/32 only and no public hostname. Only enrolled Cloudflare One devices can use it, gated by Cloudflare Gateway. It never puts step-ca on the internet. Why it exists and why it sits on [[VLAN52]]: [[stepca-remote-access-via-warp]].

OPNsense alias `pve_stepca_tunnel`. Its only internal path is `pve_int_stepca` on TCP 9000; everything else internal is blocked by the DMZTunnels block rule.

Build status and remaining steps: [[session-2026-10-17-stepca-remote-tunnel]].

## Related
- [[pve-services]]
- [[VLAN52]]
- [[pve-int-stepca]]
- [[pve-bastion]]

---
name: pve-bastion
type: lxc
vmid: 2202
node: pve-services
ip: 10.0.51.10
vlan: VLAN51
status: live
role: SSH bastion, single identity-verified SSH entry point for internal nodes
---

**Login:** `admin`, password auth. Normally accessed via `pct enter` from [[pve-services]] rather than direct SSH.

Built Sep 21, 2026 on [[pve-services]] (VMID 2202, unprivileged Debian 13, static 10.0.51.10/24, DNS 10.0.51.1). Details in [[session-2026-09-21-stepca-and-bastion]]. `nesting=1` was needed for the Debian 13 systemd credentials bug, and the `cloudflare-warp` package is installed.

**State as left in that session (confirmed by Daniel on Sep 24, 2026):** WARP device enrollment is not complete, because the final authorize redirect fails with `ERR_QUIC_PROTOCOL_ERROR`. sshd does not yet trust step-ca's SSH user CA. The temporary `workstation` to `bastion` port 22 rule is not recorded as removed. No SSH cutover has happened, so direct workstation SSH to internal nodes is not blocked.

Design: native-install LXC running the Cloudflare WARP Connector itself. Access uses two separate MFA layers: WARP enrollment via [[authentik]], then a short-lived SSH cert from [[pve-int-stepca]] via its own Authentik login. Full design and build status in [[sop-ssh-bastion-build|SSH Bastion Build]].

Tailscale is not installed on this LXC and must never be. It must not be able to reach the tailnet. See [[tailscale-access-model]].

Hostname confirmed Sep 23, 2026 as `pve-bastion`. The firewall alias is still called `bastion`.

## Related
- [[pve-services]]
- [[VLAN51]]
- [[Tailscale]]

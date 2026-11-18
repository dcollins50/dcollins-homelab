---
name: pve-bastion
type: lxc
vmid: 2202
node: pve-services
ip: 10.0.51.x
vlan: VLAN51
status: live
role: SSH bastion, single identity-verified SSH entry point for internal nodes
---

**Login:** `[admin-user]`, SSH certificate only since Oct 3, 2026 (passwords are off, see [[bastion-ssh-certificate-login]]). Way in without a certificate: `pct enter 2202` from [[pve-services]].

Built Sep 21, 2026 on [[pve-services]] (VMID 2202, unprivileged Debian 13, static 10.0.51.x/24, DNS 10.0.51.x). Details in [[session-2026-09-21-stepca-and-bastion]]. `nesting=1` was needed for the Debian 13 systemd credentials bug, and the `cloudflare-warp` package is installed.

**State as left in that session (confirmed by Daniel on Sep 24, 2026):** WARP device enrollment is not complete, because the final authorize redirect fails with `ERR_QUIC_PROTOCOL_ERROR`. sshd does not yet trust step-ca's SSH user CA. The temporary `workstation` to `bastion` port 22 rule is not recorded as removed. No SSH cutover has happened, so direct workstation SSH to internal nodes is not blocked.

**Update Oct 2, 2026:** enrollment now completes (see [[session-2026-10-02-bastion-warp-enrollment]]). The host is registered as Mesh node `bastion-node`, Mesh IP 100.x.x.x, Mesh IP only with no advertised routes. It is currently disconnected, because the WARP DNS proxy cannot bind IPv6 while IPv6 is disabled here. See [[cloudflare-warp-mesh-node-headless]]. The sshd and port 22 items above are unchanged.

**Update Oct 3, 2026:** the disconnect is fixed. IPv6 is now enabled for the WARP interface only (`all` and `eth0` stay disabled, `default` is 0). The node stayed Connected and came back connected after a container reboot. See [[session-2026-10-03-bastion-warp-ipv6]] and [[cloudflare-warp-mesh-node-headless]]. Later the same day a host firewall was added: default drop on input, SSH accepted only on the WARP interface, port 22 on `eth0` dropped, outbound open. See [[bastion-host-firewall]]. From the workstation (Mesh address 100.x.x.x) ping and SSH to 100.x.x.x work, tested Oct 3, 2026, see [[cloudflare-warp-windows-workstation-mesh-access]]. Cloudflare Gateway now allows only the admin identity to reach 100.x.x.x on TCP 22 and blocks the rest of the private ranges, tested Oct 3, 2026 (ICMP is not gated). SSH is still password auth and sshd does not yet trust the step-ca user CA.

Design: native-install LXC running the Cloudflare WARP Connector itself. Access uses two separate MFA layers: WARP enrollment via [[authentik]], then a short-lived SSH cert from [[pve-int-stepca]] via its own Authentik login. Full design and build status in [[sop-ssh-bastion-build|SSH Bastion Build]].

Tailscale is not installed on this LXC and must never be. It must not be able to reach the tailnet. See [[tailscale-access-model]].

Hostname confirmed Sep 23, 2026 as `pve-bastion`. The firewall alias is still called `bastion`.

## Related
- [[pve-services]]
- [[VLAN51]]
- [[Tailscale]]

## Update Oct 3, 2026 (later): SSH certificate login
sshd trusts the step-ca SSH user CA and password login is off. Tested from the workstation over the Mesh: certificate login works, a password only attempt is refused. Files: `/etc/ssh/step_user_ca.pub`, `/etc/ssh/auth_principals/[admin-user]`, `/etc/ssh/sshd_config.d/10-step-ca.conf` and `20-no-passwords.conf`. Use `systemctl restart ssh`, not `reload`, because sshd runs under `ssh.socket`. Details in [[bastion-ssh-certificate-login]] and [[session-2026-10-03-stepca-ssh-ca]]. The temporary OPNsense `workstation` to `bastion` port 22 rule was reported deleted by Daniel on Oct 3, 2026 (not re-tested). The earlier lines in this note about password auth and sshd not trusting step-ca are superseded.

---
name: bastion-host-firewall
type: decision
status: live
---

## Decision
[[pve-bastion]] runs its own host firewall (nftables, `table inet filter`), default drop on input and forward, open on output. Built and verified Oct 3, 2026 (Daniel chose both settings below). See [[session-2026-10-03-bastion-warp-ipv6]].

Input accepts only:
- replies to traffic the bastion started (`ct state established,related`)
- loopback
- ICMP and ICMPv6
- TCP port 22 arriving on the `CloudflareWARP` interface

Everything else inbound is dropped, including port 22 on `eth0`.

## Why
Mesh SSH bypasses the OPNsense inbound rules, so the Oct 2 design named a host firewall as one of the controls ([[session-2026-10-02-bastion-warp-enrollment]]). It did not exist until now. Before this, sshd listened on every address and WARP's own table accepted everything on the WARP interface.

## Choices made
- **Mesh only for SSH.** Port 22 on `eth0` is dropped. Way in until the workstation joins the Mesh is `pct enter 2202` from [[pve-services]]. This also makes the temporary `workstation` to `bastion` OPNsense rule useless, but that rule should still be removed.
- **Outbound left open.** OPNsense already limits outbound per alias. A second list on the host would be a second place to update when Cloudflare addresses change.

## Rules that must stay true
- No `flush ruleset` in `/etc/nftables.conf`. It would also delete `table inet cloudflare-warp`, which the WARP client manages and which must not be edited.
- The WARP interface is matched with `iifname` (by name), not `iif`, so the file still loads at boot before WARP creates the interface.
- Our table and WARP's table are separate base chains on the same hook. A drop in either one drops the packet, so WARP's accept-all rule for its interface does not override ours.
- Never stop or restart `nftables.service` on the bastion. Apply changes with `systemctl reload nftables` or `nft -f /etc/nftables.conf`.

## Changing it
1. Edit `/etc/nftables.conf` as root through `pct enter 2202`. A backup of the old default file is `/etc/nftables.conf.bak-2026-10-03`.
2. Check it: `nft -c -f /etc/nftables.conf`.
3. Apply it live: `nft -f /etc/nftables.conf`. The file deletes and recreates only `table inet filter`. `systemctl reload nftables` does the same thing.
4. Check WARP: `su - [admin-user] -c "warp-cli status"`.
5. Test a reboot before trusting it, because a live apply proves nothing about boot.
Revert in an emergency: `nft delete table inet filter` removes the table and leaves the host open.

## Verified
- Check, live apply, and persistent file all done. After `pct reboot 2202`: both tables present, input chain `policy drop` with the five rules, WARP Connected and healthy.
- From the workstation over the Mesh: ping and SSH to port 22 on 100.x.x.x work (Oct 3, 2026). `SSH_CONNECTION` showed 100.x.x.x to 100.x.x.x.
- `systemctl cat nftables` on the bastion shows `ExecStop=/usr/sbin/nft flush ruleset`, while `ExecStart` and `ExecReload` both run `nft -f /etc/nftables.conf`. So stopping or restarting the service deletes every table, including WARP's. Reload is safe because the file replaces only `table inet filter`.

## Cloudflare Gateway layer (added later on Oct 3)
A second control in front of the host firewall. Network policies, in this order: 1 "Allow admin SSH to bastion" (Allow, Destination IP `100.x.x.x`, Destination Port `22`, User Email `user@example.com`), 2 "Default deny for private traffic" (Block, enabled). The order matters, because a deny at number 1 would block the Allow. Tested by switching the Allow off (new SSH connections reset) and back on (they work). ICMP is not covered by network policies. Details in [[session-2026-10-03-bastion-warp-ipv6]].

## Not verified
- Cloudflare layer scope: only TCP port 22 was tested with the Allow policy on and off. Other ports, UDP and other users were not tested. ICMP is not filtered by network policies.
- That nothing else on the bastion needs unsolicited inbound traffic. Nothing was seen listening except sshd, the WARP DNS proxy on loopback and mail on loopback.

## Related
- [[pve-bastion]]
- [[cloudflare-warp-mesh-node-headless]]
- [[sop-ssh-bastion-build|SSH Bastion Build]]

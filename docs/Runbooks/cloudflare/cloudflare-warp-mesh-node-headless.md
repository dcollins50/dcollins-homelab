---
name: cloudflare-warp-mesh-node-headless
type: runbook
tool: cloudflare
---

# Runbook: Headless WARP on Linux, enrolling as a Mesh node

**When to use:** A headless Linux box (no browser) must join Cloudflare Zero Trust. Built from the [[pve-bastion]] work on Oct 2, 2026. Read [[cloudflare-warp-zero-trust-enrollment]] first for the IdP and device enrollment policy prerequisites.

**Status:** Verified on [[pve-bastion]] on Oct 3, 2026: token hand-off, node conversion, and a connection that stayed up and came back after a container reboot with the IPv6 fix below. Reaching the node from another device over the Mesh is not tested yet.

## Two kinds of enrollment
- **User device:** `warp-cli registration new <team>`, authenticates a person through the IdP. This is what the first bastion attempt used.
- **Mesh node:** `warp-cli connector new <TOKEN>`, token created in the dashboard under the Mesh page. This is the right type for a server that other devices reach. The bastion was enrolled as a user device first, then converted: `warp-cli registration delete`, then `connector new`. It shows in the dashboard as node `bastion-node` with Mesh IP 100.x.x.x (Mesh range 100.96.0.0/12).

## Headless user-device enrollment (token hand-off)
Clicking "Open in Cloudflare One Client" does nothing on a headless host. Instead:
1. As a non-root user on the host: `warp-cli registration new <team>`.
2. Open the printed URL in a browser on another machine and log in through the IdP.
3. On the final success page, view the page source and copy the token URL. It is valid for about 60 seconds.
4. On the host: `warp-cli registration token "<token url>"`, quoted.
5. Check with `warp-cli registration show`.
Do not paste the token anywhere except the host. If it expires, or `warp-cli` reports `ApiMismatch` or "Does not exist in API", delete the registration and do exactly one clean `registration new`.

## Convert to a Mesh node
```bash
warp-cli registration delete
warp-cli --accept-tos connector new <TOKEN>
warp-cli --accept-tos connect
```
Container needs `/dev/net/tun` available.

## Known problem: connect loops when IPv6 is disabled
The build pattern disables IPv6 in every LXC ([[proxmox-lxc-build-pattern]]). The WARP local DNS proxy then fails to bind and the client flaps between Connecting and Disconnected:
```
OnomaDnsServerError(SocketBind { addr: [2606:4700:cf1:2000::1]:53 ... AddrNotAvailable })
```
The WARP DNS proxy binds an IPv6 address on the WARP interface itself, so the fix is to allow IPv6 on that interface and nowhere else.

## Fix applied Oct 3, 2026: IPv6 on the WARP interface only
Keep `all` and `eth0` disabled, and enable IPv6 for interfaces created later (the WARP interface is created after boot). Put this in both `/etc/sysctl.d/99-disable-ipv6.conf` and `/etc/sysctl.conf`:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.eth0.disable_ipv6 = 1
```
- The order matters. Writing `all` also rewrites `default` and every existing interface, so `default = 0` must come after the `all` line.
- Both files need it. `/etc/sysctl.conf` is read last, so a leftover `default = 1` there silently undoes the other file.
- Anything else created in this container after boot also gets IPv6 with this setting. Fine on the bastion, which only has `eth0` and the WARP interface.
- Test it live first with `echo 0 > /proc/sys/net/ipv6/conf/default/disable_ipv6` as root, then `warp-cli connect`.
- Verify after a reboot with `grep -H . /proc/sys/net/ipv6/conf/{all,default,eth0}/disable_ipv6`. Expected values are 1, 0, 1.
- What was seen: `CloudflareWARP` got two global IPv6 addresses (including `2606:4700:cf1:2000::1`), `eth0` and `lo` stayed IPv4 only, `warp-cli status` showed Connected and healthy, and it reconnected on its own after `pct reboot`.

Not applied: a Traffic only device profile, and a carve-out that gives WARP its own local IPv6 address. Neither was needed.

## Exposure after the fix
This does not add a new boundary. IPv4 on the WARP interface was already accepted the same way. On the bastion today: sshd listens on `*:22`, `table inet filter` is empty with `policy accept`, and WARP's own `table inet cloudflare-warp` accepts all input on the `CloudflareWARP` interface. Reach over the Mesh is limited only by Cloudflare's Mesh policies and sshd. A host firewall was added later the same day in its own nftables table, because WARP manages its table. See [[bastion-host-firewall]].

## Unverified
- Whether Mesh peers can reach the node over its IPv6 addresses.
- Whether "WARP Connector" and "Mesh node" are the same product.
- How "Traffic only" service mode behaves for a node.
- Whether the default Mesh policies limit access to SSH only. Check before trusting them.

## Related Documentation
- [[cloudflare-warp-zero-trust-enrollment]]
- [[npm-fix-502-upstream-header-too-big]]
- [[sop-ssh-bastion-build|SSH Bastion Build]]

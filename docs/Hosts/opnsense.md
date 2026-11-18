---
name: opnsense
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.x
vlan: all (firewall/router)
status: live
role: Dedicated firewall, IDS/IPS, VLAN routing, NAT
---

HP EliteDesk G3. Standalone unit, NOT part of the Proxmox cluster (never count as a fifth node).

**Version:** OPNsense 26.1.11_10, patched Oct 4, 2026. Upgraded from 25.7.11_9 (off the end-of-life 25.7 series) to 26.1.2 that same day, then patched within the 26.1 series to 26.1.11_10 hours later. See [[opnsense-major-upgrade]]. FreeBSD version not rechecked since the 25.7 patch.
**WAN:** connected to [[heimdall]] eth0

Suricata runs on the WAN interface in detection-only (IDS) mode; EVE JSON logs ship to Logstash → Elasticsearch. Promotion to inline blocking (IPS) is a pending hardening item.

Update policy: checked roughly every 2 weeks, deliberately held back if a known unpatched vulnerability exists in the new release, applied once a fix ships.

All inter-VLAN routing and firewall enforcement runs here. No inbound port forwarding on WAN — all external access via WireGuard, Cloudflare Tunnel, or Tailscale.

## DNS (Unbound), as of Oct 31, 2026
Unbound serves every VLAN gateway and forwards everything it does not own to Pi-hole on [[heimdall]] with `forward-first`, so a dead Pi-hole falls back to normal recursion (tested, first lookup about 12 s slower). Custom config lives in `/usr/local/etc/unbound.opnsense.d/` (copied into the chroot `/var/unbound/etc/` on restart). Current custom files: `dot.conf` (generated, `homelab.local` forward) and `50-pihole-catchall.conf` (`forward-zone` for `.` to 192.168.100.x). Restart with `sudo configctl unbound restart`. `sudo unbound-checkconf /var/unbound/unbound.conf` fails on a Python DNSBL module path, so it cannot validate changes; see [[dns-phase1-consolidation]] for the throwaway-config workaround. Blocked domains return an empty answer through Unbound (Pi-hole itself returns `0.0.0.0`). DHCP is served on VLAN40 (`10.99.0.x`). As a non-root user `pgrep` and `sockstat` show nothing, use `sudo`.

## Related
- [[heimdall]]
- All VLAN notes in Network/

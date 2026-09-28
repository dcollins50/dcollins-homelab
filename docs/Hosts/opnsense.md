---
name: opnsense
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.1
vlan: all (firewall/router)
status: live
role: Dedicated firewall, IDS/IPS, VLAN routing, NAT
---

HP EliteDesk G3. Standalone unit, NOT part of the Proxmox cluster (never count as a fifth node).

**Version:** OPNSense 25.7 / FreeBSD 14.3
**WAN:** connected to [[heimdall]] eth0

Suricata runs on the WAN interface in detection-only (IDS) mode; EVE JSON logs ship to Logstash → Elasticsearch. Promotion to inline blocking (IPS) is a pending hardening item.

Update policy: checked roughly every 2 weeks, deliberately held back if a known unpatched vulnerability exists in the new release, applied once a fix ships.

All inter-VLAN routing and firewall enforcement runs here. No inbound port forwarding on WAN — all external access via WireGuard, Cloudflare Tunnel, or Tailscale.

## Related
- [[heimdall]]
- All VLAN notes in Network/

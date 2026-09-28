---
name: heimdall
type: physical-node
vmid: n/a
node: n/a
ip: 192.168.100.1
vlan: n/a (WAN bridge)
status: live
role: WiFi bridge, WAN gateway, Pi-hole DNS, WireGuard VPN
---

**Login:** `admin`, SSH key auth (confirmed working Sep 25, 2026 during UTC rollout — previously undocumented, an earlier attempt guessed `pi` and failed).

Raspberry Pi 5. Racked alongside the cluster, runs continuously. WAN entry point for the entire environment.

Bridges WiFi uplink (wlan0) to wired interface (eth0), which connects to [[opnsense]]'s WAN port. Has a real public IPv4 (no CGNAT), so WireGuard operates without relay.

Runs Pi-hole (DNS filtering for all VLANs) and WireGuard (remote-admin VPN, secondary path behind the [[pve-bastion]] and ahead of Tailscale, still in progress; see [[remote-access-path-order]]).

## Related
- [[opnsense]]

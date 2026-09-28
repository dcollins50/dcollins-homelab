---
name: VLAN11
vlan_id: 11
subnet: 10.0.11.0/24
status: planned
role: AIops
---

Decided Sep 23, 2026. Dedicated segment for the local AI inference host and future AIops workloads. The subnet was chosen by Daniel. VLAN ID 11 follows the existing convention (the subnet's third octet matches the VLAN ID on [[VLAN10]], [[VLAN20]], [[VLAN30]], [[VLAN50]] and [[VLAN51]]) and was confirmed by Daniel on Sep 23. See [[aiops-vlan]] for the decision.

Not built yet. Build with [[opnsense-isolated-vlan|Build a New Isolated VLAN Segment]]. Gateway 10.0.11.1. Built with DHCP from the start (Kea, reservations only) as the pilot for [[dhcp-dns-rollout]]. Decided Sep 23.

## Planned hosts
| Host | IP |
|------|----|
| [[jetson-orin-nano]] | 10.0.11.10 (currently 10.99.0.100 on [[VLAN40]]) |

## Proposed firewall rules (not agreed or built)
Scoped to the jetson's host alias, default deny otherwise:
- DNS and NTP to the VLAN gateway
- [[soar-host]] to the jetson's inference API port (rule on the VLAN10 interface)
- Log shipping to soc-stack using the existing `SOC_Stack_Ingest` and `SOC_Stack_Log_Ports` aliases
- Outbound TCP 443 for Tailscale, OS packages and model pulls, to destinations outside RFC1918 only, so it cannot reach internal hosts

## Open items
- VLAN ID 11 confirmed Sep 23.
- Switch: the jetson is on port 6 (confirmed Sep 23). Port 6 moves from VLAN40 to VLAN11 untagged. The OPNSense trunk (port 5) needs VLAN11 tagged.
- The model runs under Ollama but is not being served right now (confirmed Sep 23). The API port and listen address are undecided.
- Decided Sep 23: the jetson stays on Tailscale and Daniel accepts the outbound 443 hole. Tailscale's own guidance is outbound TCP 443 to any destination, because the relay server list grows over time. UDP 3478 and 41641 outbound are optional and only improve direct connections. Allowing 443 anywhere is an exfiltration path from a host that reads untrusted text.
- Outbound access for OS packages and model updates.
- The existing VLAN40 allow rule for log shipping to soc-stack covers the whole VLAN40 net. Review it once the jetson has moved.

## Related
- [[Network]]
- [[aiops-vlan]]
- [[jetson-orin-nano]]

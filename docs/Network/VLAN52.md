---
name: VLAN52
vlan_id: 52
subnet: 10.0.52.0/24
status: live
role: DMZTunnels, Cloudflare Tunnel connector containers only
---

Created Oct 17, 2026 (named DMZTunnels). Tagged on switch trunk ports 2 (pve-services) and 5 (OPNSense LAN). OPNsense VLAN device on parent `re0`, tag 52, static gateway 10.0.52.x/24. Fully static, no DHCP. Outbound NAT is Automatic.

**Why it exists:** hosts in the same subnet talk directly and never pass through OPNsense. Putting tunnel containers on their own VLAN makes every path from a tunnel to its internal target cross OPNsense, where a scoped rule allows it and it is logged. Tunnel containers on this VLAN can still reach each other directly, which was accepted. See [[stepca-remote-access-via-warp]].

**Firewall policy:** per-host pass rules to exactly what each tunnel needs (for [[pve-stepca-tunnel]]: DNS to the gateway, `cf_edge_ipv4` on 7844 TCP/UDP and 443 TCP, `pve_int_stepca` 9000 TCP, `debian_repos` 443 TCP for apt, as of Oct 17, 2026), then "Block DMZTunnels to internal VLANs" (source DMZTunnels net, destination `RFC1918_Networks`, logged). No ICMP rule, so pinging the gateway fails by design.

## Hosts
| Host | IP |
|------|----|
| [[pve-stepca-tunnel]] (LXC 2101) | 10.0.52.x |
| [[pve-authtunnel]] (LXC 2100) | 10.0.52.x (planned move from [[VLAN50]]) |

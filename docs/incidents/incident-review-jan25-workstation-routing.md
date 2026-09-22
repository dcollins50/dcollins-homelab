# Incident: Services VLAN Web UIs Unreachable (Workstation Routing)

**Date:** 2026-01-25
**Systems affected:** Primary workstation (dual-homed), services-host (10.0.20.30), OPNSense
**Duration:** ~3 hours of troubleshooting across multiple chat sessions
**Status:** Resolved

## Summary

Every web UI on the Services VLAN (Portainer, Pi-hole, Nginx Proxy Manager, Uptime Kuma) became unreachable from the primary workstation. The services themselves were healthy the whole time. The workstation had no route for 10.0.20.0/24, so Windows sent that traffic out the WiFi adapter, where the home network had no path to the lab. Most of the time spent on this went into false leads on the server and firewall side, and one of those false leads (a firewall state reset) made things worse before the real cause was found.

## Environment at the time

| Item | Value |
|-|-|
| Workstation WiFi | 192.168.1.241/24, gateway 192.168.1.254, interface metric 35 |
| Workstation Ethernet | 10.0.0.199/24, gateway 10.0.0.1 (OPNSense LAN), interface metric 456 |
| Services VLAN | 10.0.20.0/24 |
| services-host | 10.0.20.30 |

## Timeline

1. **Start:** Portainer showed the Docker environment as "down" even though the services were reachable.
2. **Session 1:** Checked containers, Portainer server and agent logs, `curl localhost:9443`, UFW (inactive), iptables (accepting), Docker networking. Nothing wrong on the host. During this session, all web UI access to the Services VLAN from the workstation was lost.
3. **Session 2:** Worked from the assumption that OPNSense was blocking traffic.
   - Reviewed the "Allow Management to Services" rule. It was correct.
   - Added an "Allow Services to Internet" rule because Docker DNS failures showed up in journalctl. The DNS failures were a symptom, not the cause.
   - Reset OPNSense firewall states. SSH to services-host, which had been working, dropped. VNC console access was also unavailable.
4. **Session 3:** Ran `Test-NetConnection` from PowerShell. The output showed the connection leaving from `SourceAddress: 192.168.1.241` on `InterfaceAlias: Wi-Fi`. That was the root cause.
5. **Fix applied:** Added a persistent static route for 10.0.0.0/16 through the Ethernet gateway. Web UIs came back immediately.

## Root Cause

The Windows routing table had two default routes: WiFi at metric 35 and Ethernet at metric 456. The only lab-specific route was the on-link 10.0.0.0/24 on Ethernet. Nothing covered 10.0.20.0/24 or any other VLAN subnet, so Windows used the lower-metric default route through WiFi. The home network has no route to 10.0.x.x, and OPNSense's "LAN net" alias (10.0.0.0/24) does not include 192.168.1.0/24, so the traffic went nowhere.

Why it had worked before this is unknown. Possible explanations recorded at the time were a reboot, a Windows update changing interface metrics, or a network profile change. None were confirmed.

## Fix

```powershell
route ADD 10.0.0.0 MASK 255.255.0.0 10.0.0.1 METRIC 10 -p
```

`-p` makes the route persistent across reboots. Metric 10 keeps it ahead of both default routes. WiFi still handles general internet traffic and Ethernet handles everything in 10.0.0.0/16.

Verification:

```powershell
route print | Select-String "10.0.0.0"
Test-NetConnection -ComputerName 10.0.20.30 -Port 443
```

`SourceAddress` should read 10.0.0.199, not 192.168.1.241.

## Changes made during troubleshooting

| Change | Outcome |
|-|-|
| "Allow Services to Internet" rule on the Services interface | Not related to the outage. Recorded at the time as removed. |
| OPNSense firewall state reset | Dropped all established connections, including the working SSH session to services-host. Connections recovered after the routing fix. |
| services-host | No changes were made to Docker, iptables, or host networking. |

## Where the troubleshooting went wrong

- The server was reachable from other paths, which pointed at the client, but the client was never checked first.
- The `SourceAddress` in the `Test-NetConnection` output identified the problem on sight and should have been requested at the start.
- Docker DNS errors were treated as a cause instead of a symptom.
- A new firewall rule was added in the middle of an outage, which added a variable instead of removing one.
- A firewall state reset was used as a troubleshooting step. It should be a last resort.

## Lessons Learned

- When a service works from one place and not another, check the client's routing first: `route print` and `Test-NetConnection` before touching the firewall.
- A dual-homed workstation needs explicit routes for every lab subnet. Relying on default route metrics is fragile.
- Make one change at a time during an outage, and do not add new rules until the cause is known.

## Open items

- The Ethernet interface metric (456) is still much higher than WiFi. Not changed.
- Consider a "Lab_Net" / "Home_Net" alias split in OPNSense so rules document the dual-homed workstation explicitly.

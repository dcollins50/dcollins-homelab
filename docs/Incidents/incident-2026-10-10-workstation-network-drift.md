---
name: incident-2026-10-10-workstation-network-drift
type: incident
date: 2026-10-10
status: open
---

# Incident: Workstation Wi-Fi dead, dual-homed config drifted from documented state

**Date:** 2026-10-10
**Systems affected:** Primary workstation (dual-homed), AT&T gateway DHCP/device list
**Status:** Working again. Open: DHCP reservation not yet created, root cause of the drift not proven.

## Summary

The morning after a Windows cumulative update (KB5121794, installed 2026-10-09 per `Get-HotFix`), the workstation had no internet over Wi-Fi even though the adapter showed Up and connected at 91% signal. Pings to the AT&T gateway, a public IP, and the rack Pi-hole failed. Internet only worked through Ethernet 2 (the rack path), because the Ethernet default route had started winning.

Three separate problems were stacked on top of each other:

1. **Duplicate IP.** The static `192.168.1.x` on Wi-Fi was marked `Duplicate` by Windows. Another Wi-Fi client with a randomized MAC (`xx:xx:xx:xx:xx:xx`) was using the same address. With its static address rejected, the adapter was left with only `169.254.188.170`, which cannot reach the gateway.
2. **Interface metrics reverted to automatic.** Ethernet 2 was at 25 and Wi-Fi at 30, both automatic. The documented design has Wi-Fi winning the default route (interface metric 30) and Ethernet 2 losing it (interface metric 200). With the automatic values, Ethernet won and internet traffic went through OPNsense and Heimdall.
3. **Homelab routes missing from the active routing table.** The persistent store still held the old `/24` routes and the default route, but `route print -4` showed no homelab routes besides the default. Once Wi-Fi was fixed to win the default route, homelab subnets would have gone to the AT&T gateway.

## Desired state (the target to recover to)

| Item | Value |
|-|-|
| Wi-Fi | Intel AX210, gateway 192.168.1.x, interface metric 30 (manual), 192.168.1.x via a Fixed allocation (DHCP reservation) on the AT&T gateway for MAC xx:xx:xx:xx:xx:xx |
| Wi-Fi DNS | 192.168.100.x first, 192.168.1.x second (set manually by Daniel) |
| Ethernet 2 | Intel I225-V #2, 10.0.0.x/24, gateway 10.0.0.x (OPNsense LAN), interface metric 200 (manual) |
| Ethernet 2 DNS | 192.168.100.x, 8.8.8.8 |
| Default route | Wi-Fi wins (30), Ethernet 2 is the backup (200) |
| Persistent routes via 10.0.0.x | 10.0.0.0/16, 10.99.0.0/24, 192.168.100.0/24, plus the older /24 entries for 10.0.20.x to 10.0.60.x |
| Cloudflare One Client (WARP) | Connected, enrolled Oct 3, see [[cloudflare-warp-windows-workstation-mesh-access]] |
| Tailscale | Up |

Effective check: `route print -4` should show two default routes, `10.0.0.x` at metric 200 and `192.168.1.x` at metric 30. Note that `Get-NetRoute` shows RouteMetric 0 for both and is misleading for this.

## Timeline

1. **Oct 9, night:** KB5121794 installed (Get-HotFix shows InstalledOn 10/9/2026). I could not find public documentation of what it changes, so its role is unproven.
2. **Oct 10, 05:49, 06:01, 06:11:** Tcpip event 4199 logged an address conflict for 192.168.1.x with MAC xx:xx:xx:xx:xx:xx.
3. **About 05:56:** `ping 192.168.100.x` and `ping [public-ip]` returned "General failure". Wi-Fi Network Connection Details showed DHCP disabled, `192.168.1.x/24` and `169.254.188.170/16`.
4. `netsh winsock reset` and `netsh int ip reset` were run and the PC rebooted. No improvement. Whether they changed anything is unknown.
5. Internet worked only through Ethernet 2. `Get-NetAdapter` showed Wi-Fi Up with driver [public-ip] (2024-01-21), so a driver fault was ruled out as the first suspect.
6. `Get-NetIPConfiguration` showed DNS `127.0.2.2` and `127.0.2.3`. These are the Cloudflare WARP local DNS proxy addresses, per Cloudflare's WARP architecture docs. WARP reported healthy. Disconnecting it did not restore Wi-Fi, so WARP was ruled out.
7. `netsh wlan show interfaces` showed the adapter associated to ATTpUXbQ2j on 5 GHz. `arp -a` resolved the gateway MAC. `Get-NetIPAddress` showed `192.168.1.x` with AddressState **Duplicate**.
8. Tcpip 4199 events gave the other device's MAC. The AT&T gateway device list showed it as an unnamed Wi-Fi client, allocation "pending", last activity Fri Oct 9 23:16. Heimdall's MACs (`88:a2:9e:...`) were checked with `ip link` and ruled out.
9. Cleared and rescanned the gateway device list. The conflicting entry disappeared. The device was never identified.
10. Wi-Fi was switched to DHCP. The live Settings view then showed `192.168.1.x` while the saved profile still read Manual `.241` until edited.
11. Interface metrics set manually: Ethernet 2 to 200, Wi-Fi to 30. `route print -4` confirmed defaults at 30 (Wi-Fi) and 200 (Ethernet 2).
12. Persistent routes added: 10.0.0.0/16, 10.99.0.0/24, 192.168.100.0/24 via 10.0.0.x, metric 10. `Find-NetRoute` for 192.168.100.x, 10.0.30.x, 10.99.0.x and 10.0.10.x all resolved to Ethernet 2 via 10.0.0.x. `tracert -d 10.0.30.x` returned two hops (10.0.0.x, then the host). `Get-NetRoute -PolicyStore PersistentStore` confirmed the routes persist.
13. DNS check: `192.168.100.x` answered google.com in about 21 ms and opnsense.homelab.local in about 14 ms.
14. WARP was disconnected for the test and was turned back on afterward.
15. The gateway's IP Allocation page showed the unknown device (`xx:xx:xx:xx:xx:xx`) as a normal DHCP Allocation on `192.168.1.x`, status on, and the workstation as DHCP on `192.168.1.x`. So `.241` was handed out from the gateway's DHCP pool.
16. The house Pi-hole network overview confirmed the workstation (`xx:xx:xx:xx:xx:xx`) had been seen on `.241` since 2025-10-05, then on `.242`. The unknown MAC was not visible in the captured rows.
17. Created Fixed allocations on the gateway: the unknown device to `192.168.1.x`, the workstation to `192.168.1.x`. Renewing the PC at 09:20 returned `.241` but Windows marked it Duplicate again (Tcpip 4199 at 09:18:52, same MAC). The device had kept its old lease until it renewed.
18. The device later answered a ping at `192.168.1.x` (TTL 64, 33 to 231 ms). After `Restart-NetAdapter` and a renew, the PC showed `192.168.1.x`, PrefixOrigin Dhcp, AddressState Preferred.

## Not verified

- **What caused the drift.** Candidates are the update and the stack reset run during troubleshooting. They cannot be separated from the evidence. The Jan 25 incident listed a Windows update changing interface metrics as an unconfirmed possibility too, see [[incident-2026-01-25-workstation-routing]].
- **What the device is.** A randomized-MAC (`02:`) Wi-Fi client, still unidentified. Clues only: TTL 64 (not a default Windows host), no NetBIOS name from `nbtstat`, variable latency typical of a Wi-Fi device in power save. The gateway list gives no hostname. It is now parked at `192.168.1.x`.
- **Why `.241` was available.** The gateway leased it from its DHCP pool, because the PC's static address was never reserved. The pool range itself was not checked.
- **Why the homelab routes were missing from the active table** while present in the persistent store.
- **Reachability of every subnet.** Routing was checked for four destinations and a traceroute was run for VLAN30 only. Port tests were deliberately not used because OPNsense rules can block them. See [[opnsense]].

## Open items

- [x] DHCP reservation created: Wi-Fi MAC `xx:xx:xx:xx:xx:xx` fixed to 192.168.1.x on the AT&T gateway (Home Network > IP Allocation > Allocate). Unknown device fixed to 192.168.1.x.
- [ ] Identify the device at 192.168.1.x (Pi-hole query log for that IP, Wi-Fi MAC `xx:xx:xx:xx:xx:xx` on phones and tablets, or toggle devices one at a time).
- [ ] Reboot test: confirm metrics, the default routes and the homelab routes survive.
- [ ] If the `.241` conflict returns, record the 4199 MAC, the gateway device list IP and Last Activity before clearing anything.
- [x] Decided: Wi-Fi stays on DHCP with a reservation for 192.168.1.x (the old static address), so the gateway will not lease it out again.
- [ ] Review whether VLAN41's subnet and the lab range (`10.99.0.x` vs `10.0.40.x`) are covered by the persistent routes.

## Lessons

- Use `route print -4` for effective metrics. `Get-NetRoute` hides the interface metric.
- Check the client side first: `Get-NetIPAddress` (look at AddressState), Tcpip 4199 events, and `route print -4`, before resets.
- "General failure" on ping meant Windows could not send, which pointed at the adapter's IP state, not the router.
- Capture a known-good baseline so a drift like this takes minutes to diff. See [[windows-restore-dual-homed-network]].
- OPNsense rules can make port tests misleading. Test routing with `Find-NetRoute` and `tracert -d`.

## Related

- [[windows-restore-dual-homed-network]]
- [[incident-2026-01-25-workstation-routing]]
- [[cloudflare-warp-windows-workstation-mesh-access]]
- [[Network]]
- [[opnsense]]
- [[VLAN20]]

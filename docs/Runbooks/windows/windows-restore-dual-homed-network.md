---
name: windows-restore-dual-homed-network
type: runbook
tool: windows
---

# Restore the dual-homed workstation network

Use this when the primary workstation loses internet, loses the homelab, or sends internet traffic through the rack. Built from [[incident-2026-10-10-workstation-network-drift]] and [[incident-2026-01-25-workstation-routing]]. Admin PowerShell throughout.

## Target state

| Item | Value |
|-|-|
| Wi-Fi | gateway 192.168.1.x, interface metric 30 (manual), carries internet |
| Ethernet 2 | 10.0.0.x/24, gateway 10.0.0.x (OPNsense LAN), interface metric 200 (manual), carries the homelab |
| Default routes in `route print -4` | 192.168.1.x at 30, 10.0.0.x at 200 |
| Persistent routes via 10.0.0.x | 10.0.0.0/16, 10.99.0.0/24, 192.168.100.0/24 |
| DNS | 192.168.100.x then 192.168.1.x on Wi-Fi |
| WARP and Tailscale | Both up |

## Before changing anything: capture the current state

Takes under a minute and gives you something to diff against.

```powershell
$d = "$env:USERPROFILE\Documents\net-capture-$(Get-Date -Format yyyyMMdd-HHmm)"
New-Item -ItemType Directory $d | Out-Null
ipconfig /all > "$d\ipconfig.txt"
route print -4 > "$d\route-print.txt"
Get-NetIPAddress -AddressFamily IPv4 | Format-Table IPAddress, InterfaceAlias, PrefixOrigin, AddressState > "$d\ipaddress.txt"
Get-NetIPInterface -AddressFamily IPv4 | Format-Table InterfaceAlias, AutomaticMetric, InterfaceMetric, Dhcp > "$d\interfaces.txt"
Get-NetRoute -PolicyStore PersistentStore | Format-Table DestinationPrefix, NextHop, RouteMetric, InterfaceAlias > "$d\persistent-routes.txt"
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Tcpip'; Id=4199} -MaxEvents 5 | Format-List TimeCreated, Message > "$d\tcpip-4199.txt"
```

## Step 1: Triage (about 1 minute)

```powershell
Get-NetIPAddress -InterfaceAlias "Wi-Fi" -AddressFamily IPv4 | Format-Table IPAddress, PrefixOrigin, AddressState
route print -4
```

| Symptom | Likely cause | Go to |
|-|-|-|
| Wi-Fi has a 169.254.x.x address, or its address shows `Duplicate` | IP conflict | Step 2 |
| "General failure" on ping | Adapter has no usable IP, see above | Step 2 |
| Internet works only through the rack | Ethernet wins the default route | Step 3 |
| Homelab subnets unreachable, Wi-Fi fine | Routes missing from the active table | Step 4 |
| Everything pings but names fail | DNS, or WARP state | Step 5 |

## Step 2: Fix an address conflict

1. Get the other device's MAC:
   ```powershell
   Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Tcpip'; Id=4199} -MaxEvents 3 | Format-List TimeCreated, Message
   ```
2. On the AT&T gateway at 192.168.1.x, open the device list. Note the IP, allocation type and Last Activity for that MAC before clearing anything.
3. Get the workstation online: Settings > Network and internet > Wi-Fi > the network > IP assignment > Edit > Automatic (DHCP). Editing the saved profile matters, because the live adapter can change while the saved setting still reads Manual.
4. Check:
   ```powershell
   Get-NetIPAddress -InterfaceAlias "Wi-Fi" -AddressFamily IPv4 | Format-Table IPAddress, PrefixOrigin, AddressState
   ```
   You want one 192.168.1.x address, PrefixOrigin `Dhcp`, no 169.254.x.x.
5. Make it stick with a DHCP reservation for MAC `xx:xx:xx:xx:xx:xx`. On the AT&T gateway (ARRIS firmware, confirmed on Oct 10): Home Network > IP Allocation > click Allocate next to the client > choose a Fixed address > Save. Community reports say the gateway allows only 17 reservations.
6. If another device already holds the address you want, reserve a different Fixed address for that device first. It keeps its old lease until it renews or reconnects, so the PC can get the address and Windows can still mark it `Duplicate` until then. The gateway's status column can be stale. Confirm with a new Tcpip 4199 event, then run `Restart-NetAdapter -Name "Wi-Fi"` and `ipconfig /renew "Wi-Fi"` once the other device has moved.

## Step 3: Restore the interface metrics

```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -AutomaticMetric Disabled -InterfaceMetric 200
Set-NetIPInterface -InterfaceAlias "Wi-Fi" -AutomaticMetric Disabled -InterfaceMetric 30
route print -4
```

Check `route print -4`, not `Get-NetRoute`. `Get-NetRoute` shows RouteMetric 0 for both defaults and hides the interface metric. You want 192.168.1.x at 30 and 10.0.0.x at 200.

## Step 4: Restore the homelab routes

Check the persistent store, then the active table:

```powershell
Get-NetRoute -PolicyStore PersistentStore | Format-Table DestinationPrefix, NextHop, RouteMetric, InterfaceAlias
route print -4
```

If any are missing, add them (`-p` makes them survive a reboot):

```powershell
route ADD 10.0.0.x MASK 255.255.0.0 10.0.0.x METRIC 10 -p
route ADD 10.99.0.x MASK 255.255.255.0 10.0.0.x METRIC 10 -p
route ADD 192.168.100.x MASK 255.255.255.0 10.0.0.x METRIC 10 -p
```

The /16 covers every 10.0.x.x VLAN. Older /24 entries (10.0.20.x to 10.0.60.x) may also exist and are harmless.

## Step 5: DNS and WARP

```powershell
Resolve-DnsName google.com -Server 192.168.100.x -DnsOnly
Resolve-DnsName opnsense.homelab.local -Server 192.168.100.x -DnsOnly
warp-cli status
```

When WARP is connected, Windows shows `127.0.2.2` and `127.0.2.3` as DNS servers. That is the WARP local DNS proxy and is expected. To rule WARP out, run `warp-cli disconnect`, retest, then `warp-cli connect`.

## Step 6: Verify (routing, not ports)

Port tests can fail because of OPNsense rules, so check routing separately from the firewall.

```powershell
Find-NetRoute -RemoteIPAddress 192.168.100.x | Format-Table InterfaceAlias, NextHop, RouteMetric
Find-NetRoute -RemoteIPAddress 10.0.30.x | Format-Table InterfaceAlias, NextHop, RouteMetric
Find-NetRoute -RemoteIPAddress 8.8.8.8 | Format-Table InterfaceAlias, NextHop, RouteMetric
tracert -d 10.0.30.x
```

Expected: homelab destinations go out Ethernet 2 via 10.0.0.x, 8.8.8.8 goes out Wi-Fi via 192.168.1.x, and the traceroute shows 10.0.0.x as hop 1. If the traceroute dies after hop 1, routing is fine and OPNsense or the host is dropping traffic: check Firewall > Log Files > Live View in [[opnsense]], source 10.0.0.x.

## Do not

- Do not start with `netsh winsock reset` or `netsh int ip reset`. They were run during the Oct 10 incident without helping, and whether they changed anything is unknown. Capture the config first.
- Do not reset OPNsense firewall states to troubleshoot a client problem. That broke a working SSH session during the Jan 25 incident.
- Do not add firewall rules mid-outage before the client side is checked.

## Finish

Reboot once, rerun Step 3 and Step 4 checks, and confirm nothing reverted.

## Related

- [[incident-2026-10-10-workstation-network-drift]]
- [[incident-2026-01-25-workstation-routing]]
- [[cloudflare-warp-windows-workstation-mesh-access]]
- [[Network]]
- [[opnsense]]

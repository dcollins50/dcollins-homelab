---
name: aiops-vlan-build
type: project
status: planned
---

Build plan for [[VLAN11]] and the move of [[jetson-orin-nano]] onto it. Decision and reasoning in [[aiops-vlan]]. Planned Sep 23, 2026, to be implemented later. Nothing here has been built.

A GitHub-ready version of this plan is published at `docs/change-management/aiops-vlan-build.md` in the repo.

Follows [[opnsense-isolated-vlan|Build a New Isolated VLAN Segment]] and [[opnsense-add-vlan-interface|Add a VLAN Interface]]. Decided Sep 23: VLAN11 is built with DHCP from the start and is the pilot for [[dhcp-dns-rollout]]. The jetson gets a Kea reservation for 10.0.11.10, not a static address. That makes Kea enabled on OPNsense (for this interface only) a prerequisite.

## Order matters
Do all of OPNsense first, then the switch, then the jetson, so the jetson never loses its path mid-move. Have local console access to the jetson in case the address change goes wrong.

## 1. OPNsense
1. Create the VLAN device: tag 11, parent `re0`, description AIops. The menu is Interfaces, Devices, VLAN, or Interfaces, Other Types, VLAN, depending on version.
2. Assign it as a new interface and enable it. Static IPv4 10.0.11.1/24, no gateway. Save and Apply. Then enable Kea DHCPv4 on this interface only, with a reservation for the jetson's MAC at 10.0.11.10 and no dynamic pool. Verify Kea accepts a subnet without a pool. If it requires one, use a small one and record it.
3. Aliases: a host alias for 10.0.11.10 (for example `jetson_aiops`). Reuse `SOC_Stack_Ingest` and `SOC_Stack_Log_Ports`. Check Firewall, Aliases for an existing RFC1918 alias before creating one.
4. Rules on the new interface tab, source `jetson_aiops`, each with a plain description:
   - Pass TCP/UDP 53 to 10.0.11.1.
   - Pass UDP 123 to 10.0.11.1, only if OPNsense runs its own NTP (check Services, Network Time).
   - Pass to `SOC_Stack_Ingest` on `SOC_Stack_Log_Ports`, logging on.
   - Pass TCP 443 with the destination set to the RFC1918 alias inverted. This allows Tailscale and model pulls but not internal hosts. Accepted risk: see [[aiops-vlan]].
   - Last, an explicit Block rule with logging enabled, because an unlogged block delayed the Sept 14 diagnosis (see [[incident-2026-09-13-elastic-log-shipping-outage|Elastic log shipping outage]]).

## 2. Switch (TL-SG108E)
Add VLAN 11. Tag it on port 5 (OPNsense trunk). Put port 6 untagged in VLAN 11, remove it from VLAN 40, set port 6 PVID to 11. The jetson loses connectivity here.

## 3. Jetson
Note the jetson's MAC address first (`ip link`) and create the Kea reservation before the move. After the switch change, set its connection to automatic (DHCP). Gateway and DNS arrive from DHCP and should be 10.0.11.1. The jetson's logs show NetworkManager, so `nmcli` is likely the tool. Run `nmcli con show` first. Filebeat keeps sending to 10.0.10.10 on port 5044 through the new rule.

## 4. Tests, from the jetson
- Ping 10.0.11.1 and resolve a name.
- `nc -zv 10.0.10.10 5044` succeeds.
- `curl -I https://login.tailscale.com` succeeds and `tailscale status` shows connected.
- `curl -m 5 https://10.0.20.30` fails, confirming the RFC1918 exclusion.

## 5. Afterwards
- Review the old VLAN40 log-shipping allow rule. It covers the whole VLAN40 net.
- Update [[VLAN11]], [[jetson-orin-nano]], [[switch-tl-sg108e]] (VLAN list) and the ASCII topology in [[Network]].
- Serving the model later needs a rule from [[soar-host]] to the jetson's API port. Verify Ollama's listen address and authentication behaviour against its docs first.

## Related
- [[aiops-vlan]]
- [[VLAN11]]
- [[jetson-orin-nano]]
- [[Tailscale]]
- [[dhcp-dns-rollout]]
- [[aiops-assistant-rollout]]

---
name: aiops-vlan
type: decision
status: live
---

## Decision
A new dedicated VLAN, 10.0.11.0/24, for AIops ([[VLAN11]]). The [[jetson-orin-nano]] moves out of [[VLAN40]] onto it. Decided Sep 23, 2026. Daniel never had a fixed plan for where the jetson should live and wants it where it is useful.

## Address and Tailscale
The jetson's address will be 10.0.11.10, as a DHCP reservation. VLAN11 is the pilot for [[dhcp-dns-rollout]]. It stays on Tailscale, so the VLAN allows it outbound TCP 443. Daniel accepted that this is an exfiltration path from a host that reads untrusted text (Sep 23, 2026). Proposed, not yet agreed: the 443 rule excludes RFC1918 destinations so it cannot reach internal hosts.

## Alert routing
It is only Daniel for now. If this system is provisioned for another user or org, alert routing has to be configurable per recipient.

## Reasoning discussed
- [[VLAN40]] is flat with Kali, Metasploitable2 and DVWA, with no firewall between hosts on it.
- Traffic inside a VLAN is not filtered by OPNsense, so putting the jetson on [[VLAN10]] would place it beside Elasticsearch, Wazuh, Shuffle and DFIR-IRIS with nothing between them.
- The jetson will read untrusted log text, and its output may feed automation, so it should be reachable only through narrow pinholes.

## Not chosen
- Staying on VLAN40.
- Moving to VLAN10.

## Proposed, not yet agreed
- Build plain deterministic alerts first: node quorum, per-source log silence, wazuh-manager disk, shard usage, stopped VMs. The incident postmortems record all of these as undetected for days.
- The model only classifies and explains, and outputs a fixed structure. Pre-written playbooks in [[soar-host]] do any acting, from an allowlist of actions and targets.
- Shadow mode first: recommendations only, compared against what Daniel would have done.
- Candidate auto-fix allowlist from past incidents: start a stopped VM from a listed set, restart a named service once, restart a named container once.
- Never automatic: log truncation, index deletion, corosync edits, firewall, DNS or certificate changes.

## Related
- [[VLAN11]]
- [[jetson-orin-nano]]
- [[soar-host]]
- [[Tailscale]]

---
name: VLAN55
vlan_id: 55
subnet: 10.0.55.0/24
status: live
role: CI, isolates the pve-ci runner
---

Named **CI**. Its only purpose is to isolate [[pve-ci]], the only host on it.

- Gateway 10.0.55.x, an OPNsense VLAN device on `re0`
- No DHCP
- Configured and applied on the switch ([[switch-tl-sg108e]]). VLANs 53 and 54 are reserved there.

## Firewall rules (OPNsense CI interface)
- DNS to OPNsense
- Outbound internet only (private-networks alias inverted) on 443, 80, NTP 123, and Tailscale UDP 41641/3478
- Everything else hits a logged default-deny
- Inbound: LAN rule "Workstation SSH to CI" lets the workstation SSH to pve-ci

## Hosts
| Host | IP |
|------|----|
| [[pve-ci]] (VM 800) | 10.0.55.x |

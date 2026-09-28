---
name: VLAN30
vlan_id: 30
subnet: 10.0.30.0/24
status: live
role: Trust Infrastructure
---

Repurposed (Sep 14, 2026) as dedicated trust infrastructure VLAN, housing both CAs and Authentik separately from general app hosting on [[VLAN20]]. Note: the switch's actual VLAN ID for "Dev" is 30, not 6, despite OPNSense's interface naming ("vlan06") suggesting otherwise.

## Hosts
| Host | IP |
|------|----|
| [[pve-ca-root]] (VM 500) | 10.0.30.21 |
| [[pve-ca-intermediate]] (VM 501) | 10.0.30.22 |
| [[authentik]] (LXC 2201) | 10.0.30.10 |
| [[pve-int-stepca]] (LXC 511) | 10.0.30.20 |
| [[ubuntu-401]] (VM 401) | 10.0.30.40 |

[[pve-ca-root]] and [[pve-ca-intermediate]] form the internal two-tier PKI. [[authentik]] is the self-hosted IdP/SSO. [[pve-int-stepca]] runs step-ca in Docker, planned to take over Intermediate CA duties.

## SOPs
- [[sop-pki-migration-vlan30|PKI Migration to Trust Infrastructure VLAN]]
- [[sop-stepca-cutover|step-ca Cutover]]

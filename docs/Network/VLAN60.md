---
name: VLAN60
vlan_id: 60
subnet: 10.0.60.0/24
status: planned
role: Storage
---

No devices deployed yet. Planned rule: no internet access, management access restricted to [[VLAN1]] only. The NAS is delayed specifically due to budget constraints, not just deprioritized.

**OPNsense interface (confirmed Nov 16, 2026, screenshot):** "Storage", identifier `opt6`, device `vlan07`, enabled, static 10.0.60.x/24, IPv6 none. `vlan07` carries tag 60 on parent `re0` (confirmed Nov 16, 2026, Interfaces > Devices > VLAN).

**Switch:** VLAN 60 is configured on [[switch-tl-sg108e]].

## Hosts
| Host | IP |
|------|----|
| [[pve-pbs]] (VM 6000 on [[pve-env1]]) | 10.0.60.x |

## Firewall (added Nov 16, 2026 through the OPNsense API, so they live under Firewall > Rules [new])
Aliases: `pve_pbs` (10.0.60.x), `pve_nodes` (10.0.0.x-13), `proxmox_repos` (download.proxmox.com), `pbs_api` (8007), `apt_ports` (80, 443), `pbs_admin_ports` (8007, 22). Reuses `debian_repos` from the bastion build.

Storage interface (`opt6`), all pass, source `pve_pbs`:
- DNS TCP/UDP 53 to 10.0.60.x
- NTP UDP 123 to 10.0.60.x
- TCP `apt_ports` to `debian_repos`
- TCP `apt_ports` to `proxmox_repos`

LAN interface, pass, destination `pve_pbs`:
- `pve_nodes` TCP `pbs_api` (nodes back up to pve-pbs)
- workstation 10.0.0.x TCP `pbs_admin_ports` (web UI and SSH)

Still to add: pve-pbs to the Backblaze B2 endpoint on 443 only, once the bucket exists and its endpoint hostname is known. This is the one exception to VLAN60's no-internet rule besides package updates. See [[offsite-backups-pbs-b2]].

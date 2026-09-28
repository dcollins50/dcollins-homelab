---
name: pve-env1
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.12
vlan: VLAN1 (mgmt), VLAN10 (SOC), VLAN20 (services), VLAN50 (DMZ)
status: live
role: Production services, Docker workloads, SOC automation
---

HP EliteDesk G5. One of four Proxmox nodes, Corosync quorum member.

**Specs:** 6-core Intel i5-9500 @ 3.00GHz, 31.13 GiB RAM
**Storage:** 931.5GB NVMe (primary, Proxmox data pool/VM disks), 238.5GB NVMe (secondary, available)

## Hosted VMs
- [[services-host]] (VM 200, 10.0.20.30) — Docker Host 1
- [[soar-host]] (VM 602, 10.0.10.12) — Shuffle SOAR
- [[docker-host-template]] (VM 603) — clone template
- [[npm-dmz]] (LXC 2200, 10.0.50.69) Cloudflare Tunnel and public reverse proxy
- [[pve-iris]] (VM 604, 10.0.10.13) — DFIR-IRIS

## Related
- [[VLAN1]]
- [[VLAN20]]
- [[VLAN50]]

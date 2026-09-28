---
name: pve-services
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.11
vlan: VLAN1 (mgmt), VLAN30 (trust infra), VLAN50 (DMZ), VLAN51 (bastion)
status: live
role: Internal services, PKI and trust infrastructure
---

HP EliteDesk G3. One of four Proxmox nodes, Corosync quorum member.

## Hosted VMs/LXCs
- [[ubuntu-401]] (VM 401)
- [[pve-ca-root]] (VM 500)
- [[pve-ca-intermediate]] (VM 501)
- [[authentik]] (LXC 2201)
- [[pve-int-stepca]] (LXC 511)
- [[pve-bastion]] (LXC 2202)
- [[pve-authtunnel]] (LXC 2100)

## Related
- [[VLAN1]]
- [[VLAN30]]
- [[VLAN51]]

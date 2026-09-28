---
name: Infrastructure
type: moc
status: live
---

Overview of the Proxmox cluster and supporting hardware. For per-host detail, see the linked notes in the Hosts/ folder.

## Proxmox Cluster
Four HP EliteDesk mini PCs, Corosync quorum across all four. **Version:** PVE 9.1.1 / Kernel 6.17.2-1-pve. [[opnsense]] is a completely separate dedicated unit and is NOT part of the cluster (never count it as a fifth node).

## Nodes
- [[pve-gateway]] — security lab VMs, attack platform
- [[pve-services]] — internal services, PKI/trust infrastructure
- [[pve-env1]] — production services, Docker workloads, SOC automation
- [[pve-env2]] — SOC stack (ELK, Wazuh)

## Supporting hardware
- [[heimdall]] — WiFi bridge, WAN gateway, Pi-hole, WireGuard
- [[opnsense]] — firewall, IDS/IPS, VLAN routing
- [[switch-tl-sg108e]] — VLAN tagging/trunking
- [[jetson-orin-nano]] — local AI inference

## Cluster networking
All Proxmox nodes use a single Linux bridge (`vmbr0`) with VLAN-aware mode enabled; VM interfaces get VLAN tags at the bridge level. OPNSense trunks all VLANs back to the switch.

All nodes resolve DNS via Pi-hole at 192.168.100.1. soc-stack's DNS was updated via netplan to point to 10.0.10.1 for VLAN10 resolution. OPNSense Unbound forwards `homelab.local` queries to the internal DNS server.

> [!warning] Change pending
> This describes the current DNS state. [[dns-architecture]] (Sep 23, 2026) decided that Unbound on OPNsense becomes the resolver for the stack, Pi-hole stays for domain blocking only, and internal names move from `homelab.local` to `.internal`. Not started, tracked in [[dhcp-dns-rollout]].

## Storage
| Node | Storage | Size | Usage |
|------|---------|------|-------|
| pve-env1 | NVMe (primary) | 931.5GB | Proxmox data pool, VM disks |
| pve-env1 | NVMe (secondary) | 238.5GB | Available |
| pve-env2 | NVMe (passthrough to VM 600) | 250GB | Elasticsearch data |

## Known issues / pending
| Item | Status |
|------|--------|
| Standard-PC-Q35-ICH9-2009 hostname | Noisy hostname needs `hostnamectl` fix |
| Active Directory lab (attack-range target, not production identity) | Planned — Network+ is done, so no longer blocked; not started |

## Related
- [[Network]]
- [[SOC-Stack]]
- [[PKI]]
- [[Security-Lab]]

# Infrastructure

This document covers the Proxmox cluster configuration, physical node specifications, VM layout, and supporting hardware for the homelab environment.

---

## Proxmox Cluster

The cluster runs Proxmox VE across four HP EliteDesk mini PCs. Each node has a dedicated role. The cluster uses Corosync for quorum with all four nodes participating.

**Proxmox Version:** PVE 9.1.1 / Kernel 6.17.2-1-pve

OPNSense runs on a completely separate dedicated unit and is not part of the Proxmox cluster.

---

## Physical Nodes

### pve-lab (HP EliteDesk G3)

| Field | Value |
|-------|-------|
| IP | 10.0.0.10 |
| Role | Security lab VMs, attack platform |
| Primary VLAN | VLAN1 (management), VLAN40/41 (lab VMs) |

**Hosted VMs:**

| VM | ID | Network | Role |
|----|-----|---------|------|
| kali-attack | 300 | VLAN40 | Penetration testing platform |
| metasploitable2 | 301 | VLAN41 | Vulnerable target |
| dvwa | 302 | VLAN41 | Vulnerable web application |
| malware-win11 | 400 | VLAN41 | Malware analysis sandbox |

---

### pve-services (HP EliteDesk G3)

| Field | Value |
|-------|-------|
| IP | 10.0.0.11 |
| Role | Internal services, PKI infrastructure |
| Primary VLAN | VLAN1 (management), VLAN30 (services VMs) |

**Hosted VMs:**

| VM | ID | Network | Role |
|----|-----|---------|------|
| ubuntu | 401 | VLAN30 | General services |
| pve-ca-root | 500 | VLAN30 | Root CA |
| pve-ca-intermediate | 501 | VLAN30 | Intermediate CA |

---

### pve-env1 (HP EliteDesk G5)

| Field | Value |
|-------|-------|
| IP | 10.0.0.12 |
| Role | Production services, Docker workloads |
| Primary VLAN | VLAN1 (management), VLAN20 (services VMs) |
| Storage | 931.5GB NVMe (primary), 238.5GB NVMe (secondary) |

**Hosted VMs:**

| VM | ID | Network | Role |
|----|-----|---------|------|
| services-host | 200 | VLAN20 (10.0.20.30) | Docker Host 1 |
| kalshi-mm | 290 | VLAN20 (10.0.20.50) | Market maker application |
| services-host2 | 700 | VLAN20 (10.0.20.31) | Docker Host 2 |

---

### pve-SOC (HP EliteDesk G6)

| Field | Value |
|-------|-------|
| IP | 10.0.0.13 |
| Role | SOC stack — dedicated to ELK and Wazuh |
| Primary VLAN | VLAN1 (management), VLAN10 (SOC VMs) |
| RAM | 32GB total |

**Hosted VMs:**

| VM | ID | Network | Role | Specs |
|----|-----|---------|------|-------|
| soc-stack | 600 | VLAN10 (10.0.10.10) | Elasticsearch, Logstash, Kibana | 16GB RAM, 8 cores, 200GB boot, 250GB NVMe passthrough |
| wazuh-manager | 601 | VLAN10 (10.0.10.11) | Wazuh SIEM Manager | 8GB RAM, 2 cores |

pve-SOC is reserved exclusively for the SOC stack and runs no other workloads.

---

## Supporting Hardware

### Heimdall (Raspberry Pi 5)

| Field | Value |
|-------|-------|
| Role | WiFi bridge, WAN gateway, Pi-hole DNS, WireGuard VPN |
| Network | 192.168.100.1 (management), public IPv4 (real, no CGNAT) |
| WAN | wlan0 bridged to eth0 |
| DNS | Pi-hole filtering all VLANs |
| VPN | WireGuard gateway for remote administration |

Heimdall is racked alongside the cluster and runs continuously. It is the WAN entry point for the entire environment. Because it has a real public IPv4, WireGuard operates without relay.

---

### OPNSense Firewall (HP EliteDesk G3)

| Field | Value |
|-------|-------|
| Role | Dedicated firewall, IDS/IPS, VLAN routing, NAT |
| WAN | Connected to Heimdall eth0 |
| Version | OPNSense 25.7 / FreeBSD 14.3 |
| IDS/IPS | Suricata on WAN interface, detection-only mode |

OPNSense is a completely standalone unit. It is not virtualized and is not part of the Proxmox cluster. All inter-VLAN routing and firewall enforcement runs here.

---

### TP-Link TL-SG108E (Managed Switch)

| Field | Value |
|-------|-------|
| Role | VLAN tagging and trunking |
| VLANs | 1, 10, 20, 30, 40, 41, 50, 51, 60 |

---

### Jetson Orin Nano

| Field | Value |
|-------|-------|
| Role | AI inference node |
| Network | VLAN40 (10.99.0.100) |
| Model | Qwen 2.5 3B |
| Physical | Racked alongside the cluster |

---

## Cluster Networking

All Proxmox nodes use a single Linux bridge (`vmbr0`) with VLAN-aware mode enabled. VM network interfaces are assigned VLAN tags at the bridge level. OPNSense trunks all VLANs back to the switch.

### Node DNS

All nodes resolve via Pi-hole at 192.168.100.1. SOC stack DNS was updated via netplan to point to 10.0.10.1 for VLAN10 resolution. OPNSense Unbound forwards `homelab.local` queries to the internal DNS server.

---

## Storage

| Node | Storage | Size | Usage |
|------|---------|------|-------|
| pve-env1 | NVMe (primary) | 931.5GB | Proxmox data pool, VM disks |
| pve-env1 | NVMe (secondary) | 238.5GB | Available |
| pve-SOC | NVMe (passthrough to VM 600) | 250GB | Elasticsearch data |

---

## Known Issues and Pending Items

| Item | Status |
|------|--------|
| Root SSH login on Proxmox nodes | Not yet disabled |
| Standard-PC-Q35-ICH9-2009 hostname | Noisy hostname needs `hostnamectl` fix |
| VM 700 Wazuh agent | Deferred — VM was shut down at rollout time |
| Active Directory domain controller | Planned — deferred until after Network+ exam |

---

## Related Documentation

- [Network Architecture](network.md)
- [SOC Stack](soc-stack.md)
- [Internal PKI](pki.md)
- [Security Lab](security-lab.md)

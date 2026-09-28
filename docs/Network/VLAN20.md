---
name: VLAN20
vlan_id: 20
subnet: 10.0.20.0/24
status: live
role: Services
---

**Firewall policy:** outbound internet permitted (HTTP/HTTPS via alias). DNS restricted to Heimdall's Pi-hole (192.168.100.1) as sole resolver. NTP outbound permitted. Wazuh agents reach [[wazuh-manager]] on TCP/UDP 1514-1515. Internal NPM has specific allowlisted reverse-proxy targets: Kibana (10.0.10.10:5601), Proxmox WebUI (all nodes, port 8006), services-host itself, [[pve-iris]] (443), [[authentik]] (9000, SSO reverse proxy). This VLAN can reach Authentik directly (port 9000) and [[soar-host]] (HTTP/HTTPS). Explicit block on Services-to-internal-VLANs beyond these allowlisted paths (RFC1918 block); general outbound internet still permitted.

## Hosts
| Host | IP |
|------|----|
| [[services-host]] (VM 200) | 10.0.20.30 |

Running on services-host: Vaultwarden, internal NPM, Gitea, Stremio (planned removal), Portainer, Uptime Kuma.

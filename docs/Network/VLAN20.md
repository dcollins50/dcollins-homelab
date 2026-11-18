---
name: VLAN20
vlan_id: 20
subnet: 10.0.20.0/24
status: live
role: Services
---

**Firewall policy:** outbound internet permitted (HTTP/HTTPS via alias). DNS: the rules allow Heimdall's Pi-hole (192.168.100.x) and This Firewall (any port, so Unbound on 10.0.20.x too), then block port 53 to anywhere else. Checked in the rule list Oct 31, 2026. This line previously said Pi-hole was the sole resolver, which the rules do not enforce. Ansible hosts have a rule for DNS to 10.0.20.x only. NTP outbound permitted. Wazuh agents reach [[wazuh-manager]] on TCP/UDP 1514-1515. Internal NPM has specific allowlisted reverse-proxy targets: Kibana (10.0.10.x:5601), Proxmox WebUI (all nodes, port 8006), services-host itself, [[pve-iris]] (443), [[authentik]] (9000, SSO reverse proxy). This VLAN can reach Authentik directly (port 9000) and [[soar-host]] (HTTP/HTTPS). Explicit block on Services-to-internal-VLANs beyond these allowlisted paths (RFC1918 block); general outbound internet still permitted.

## Hosts
| Host | IP |
|------|----|
| [[services-host]] (VM 200) | 10.0.20.x |

Running on services-host: Vaultwarden, internal NPM, Gitea, Stremio (planned removal), Portainer, Uptime Kuma.

**DNS (Oct 31, 2026):** [[services-host]] resolves through Unbound at `10.0.20.x`, moved from Pi-hole as part of [[dns-phase1-consolidation]]. No firewall change was needed, the existing This Firewall rule covers it.

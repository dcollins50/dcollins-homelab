---
name: services-host
type: vm
vmid: 200
node: pve-env1
ip: 10.0.20.x
vlan: VLAN20
status: live
role: Docker Host 1
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Running: Vaultwarden, internal NPM (Nginx Proxy Manager, separate from npm-dmz), Gitea, Stremio (planned removal — Stremio now runs from the workstation directly instead), Portainer, Uptime Kuma.

Uptime Kuma here is NOT configured to monitor TLS certificate expiry (Daniel, Oct 29, 2026). This note previously said it monitored TLS validity for all internal HTTPS endpoints and alerted on approaching expiry; that was wrong or out of date. Nothing currently warns before an internal cert expires.

Wazuh Docker wodle enabled; the `wazuh` user is in the `docker` group so container events can be collected without running the agent as root.

## Update Oct 31, 2026: resolver moved to Unbound
DNS on `enp6s18` now points at `10.0.20.x` (Unbound on the OPNsense VLAN20 gateway). Before: `192.168.100.x` (Pi-hole) only. Set in the single netplan file under `/etc/netplan/` (`nameservers: addresses`, written by subiquity, cloud-init `disabled`). Backup: `~/netplan-services-host.bak-2026-10-31`. Rollback: copy the backup over the file and `sudo netplan apply`. Applied with `netplan try` at about 10:31. Checked: `resolvectl status enp6s18` shows `10.0.20.x`, and `getent hosts` returned correct answers for `stepca.homelab.internal` (10.0.30.x), `authentik.homelab.example` (10.0.20.x) and google.com. `tailscale0` keeps its own MagicDNS (100.100.100.100). Docker containers here likely still use the old Pi-hole path until they restart (the same thing seen on [[authentik]]); not checked. This host loses Pi-hole blocking until Stage 4b of [[dns-phase1-consolidation]]. A Vaultwarden encrypted export was made first, per Daniel.

## Internal NPM proxy hosts (recovered Oct 31, 2026 from past chats)
No inventory of NPM's proxy hosts existed in the vault. This list comes from an earlier chat session that listed every NPM proxy host with its forward target (while scoping the Services RFC1918 firewall rule, dated around Mar 30, 2026), plus the Sep 2026 Authentik and IRIS additions. It is not read from NPM today, so it may have drifted.

| Name | Forwards to |
|---|---|
| elasticsearch.homelab.local | 10.0.10.x:5601 (Kibana, scheme https, Websockets on) |
| env1.homelab.local | 10.0.0.x:8006 |
| env2.homelab.local | 10.0.0.x:8006 |
| gateway.homelab.local | 10.0.0.x:8006 |
| services.homelab.local | 10.0.0.x:8006 |
| gitea.homelab.local | 10.0.20.x:3000 |
| npm.homelab.local | 10.0.20.x:81 |
| portainer.homelab.local | 10.0.20.x:9443 |
| uptime.homelab.local | 10.0.20.x:3001 |
| vaultwarden.homelab.local | 10.0.20.x:8080 |
| pihole.homelab.local | 192.168.100.x:80 (Heimdall) |
| authentik.homelab.example | 10.0.30.x:9000 ([[authentik]], scheme http, custom buffer config, see the 502 runbook) |
| iris.homelab.local | [[pve-iris]] port 443 |

Schemes and Websockets per host are known only for Kibana and Authentik. Proxmox web UIs, Portainer and Gitea likely need Websockets and an https upstream where noted by port, but that is not confirmed. Access lists, redirect hosts and streams: none mentioned anywhere I found. To read them from NPM: `sudo sh -c "grep -HE 'server_name|set .(server|port|forward_scheme)|Upgrade|proxy_buffer|auth_basic|satisfy|allow |deny ' /home/admin/docker/nginx-proxy-manager/data/nginx/proxy_host/*.conf"` (path is an assumption about NPM's layout). Cert expiry dates are in [[pve-ca-intermediate]].

## Related
- [[vaultwarden]] (service note and its runbooks)
- [[pve-env1]]
- [[VLAN20]]

## Update Nov 9, 2026: internal NPM upgraded to 2.16.0
Internal NPM went from 2.13.5 to 2.16.0 after a stopped-container backup (`~/npm-backup-2026-11-09.tgz` in the admin home, delete after a few clean days). Compose file `/home/admin/docker/nginx-proxy-manager/docker-compose.yml`. Pin the tag, do not use `latest`. Rollback means restoring the backup, not only swapping the tag. See [[session-2026-11-09-stepca-intermediate-rotation]].

---
name: npm-dmz
type: lxc
vmid: 2200
node: pve-env1
ip: 10.0.50.69
vlan: VLAN50
status: live
role: Public-facing reverse proxy
---

Debian 13 LXC (VMID 2200 on [[pve-env1]]) running Docker, dedicated to public-facing DMZ traffic. cloudflared runs as a systemd service. Nginx Proxy Manager and [[ntfy]] run as Docker Compose containers. Corrected Sep 24, 2026 against [[sop-cloudflare-tunnel-implementation|the Cloudflare tunnel SOP]]; the earlier text said this was not Docker. Separate from the internal NPM instance on [[services-host]], deliberately, to keep public and internal/admin services split. Admin UI on port 81.

Handles reverse proxying from the Cloudflare Tunnel (Web) to internal services. Inbound 80/443 via Cloudflare Tunnel only, no direct WAN exposure. The tunnel (`web-tunnel`) is locally managed: its hostnames are set in the `cloudflared` config file on this container, not in the Cloudflare dashboard (confirmed in the dashboard on Sep 24, 2026, see [[cloudflare-tunnel-publish]]). Daniel decided on Sep 24, 2026 to leave it locally managed for now and not use the dashboard's Configure button to migrate it.

## Related
- [[VLAN50]]
- [[pve-env1]]
- [[ntfy]]
- [[services-host]]

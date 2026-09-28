---
name: ntfy
type: container
vmid: n/a
node: pve-env1
ip: 10.0.50.69
vlan: VLAN50
status: live
role: Push notification service
---

Runs as a Docker container inside [[npm-dmz]] (LXC 2200), not as a separate VM. It shares npm-dmz's IP (10.0.50.69), publishes host port 2586, and is reverse proxied by NPM at ntfy.homelab.example. Corrected Sep 24, 2026 against [[sop-cloudflare-tunnel-implementation|the Cloudflare tunnel SOP]].

Sits behind [[authentik]] forward-auth (proxy outpost) — the browser web UI requires login, but Uptime Kuma's automated publish calls bypass the outpost and use ntfy's own token auth directly.

Target for a Cloudflare rate-limiting rule (grouped with the npm-dmz cert + local DNS item as a priority item, Sep 14, 2026). Planned use: automated push notification for any root/admin/superuser-level sign-ins across the homelab, and Wazuh-to-ntfy alert integration. Interactive ntfy action buttons is a stretch goal.

## Related
- [[VLAN50]]
- [[npm-dmz]]
- [[authentik]]

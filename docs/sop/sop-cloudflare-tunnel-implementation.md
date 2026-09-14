# SOP: Cloudflare Tunnel Implementation (Web Tunnel + ntfy)

**Objective:** Expose select self-hosted services to the public internet without any inbound port forwarding, using Cloudflare Tunnel terminated in an isolated DMZ container, and stand up a self-hosted push-notification service (ntfy) reachable regardless of VPN state.

**Status:** Web tunnel and ntfy — complete. Admin tunnel (SSH bastion + Cloudflare Access) — not yet built; see Open Items.

---

## Architecture

```
Internet
   │
   ▼
Cloudflare Edge (Tunnel)
   │  outbound-only connection, no inbound port forward on OPNSense WAN
   ▼
npm-dmz (LXC 2200, pve-env1, VLAN50/DMZ, 10.0.50.69)
   ├── cloudflared  — tunnel daemon, systemd service
   ├── nginx-proxy-manager (Docker) — hostname-based routing, TLS termination
   └── ntfy (Docker) — self-hosted push notification server
```

**Design principle carried over from existing VLAN/DMZ SOPs:** nothing internet-facing shares a physical node or network segment with anything holding credentials or sensitive data (CA, password vault, git server, SIEM). See [Placement Decision](#placement-decision) below.

---

## 1. DMZ Container Provisioning

**Container:** `npm-dmz`, VMID `2200`
**Host node:** `pve-env1`
**OS:** Debian 13 (Trixie)
**Resources:** 1GB RAM / 2 cores / 8GB disk
**Network:** VLAN 50 (DMZ) tagged at the Proxmox layer, static IP `10.0.50.69/24`, gateway `10.0.50.1`

### Placement Decision
Considered all four Proxmox nodes:
- `pve-gateway` — **ruled out.** Hosts `malware-win11`, used regularly (not rarely) for active malware analysis. Running live malicious samples on the same physical node as the sole internet-facing box was judged an unacceptable shared blast radius, independent of VLAN isolation (hypervisor-level compromise doesn't respect VLAN boundaries).
- `pve-services` — **ruled out.** Hosts the Root/Intermediate CA. Highest-value target in the lab; no internet-facing infrastructure should share a node with it.
- `pve-env2` — **ruled out.** Hosts SOC stack and Wazuh manager. Compromise risk here directly threatens detection capability.
- `pve-env1` — **selected.** Only shares the node with `kalshi-mm` (a trading bot, lower-severity risk profile, and slated for removal). Confirmed sufficient headroom (14.4G/16.6G available at time of build).

### DNS Configuration Note
The container's DNS was initially set to Pi-hole (`192.168.100.1`) during creation, which the DMZ's own firewall rules block (RFC1918 destinations are blocked outbound from DMZ by design). This caused `apt` to hang for minutes per operation. **Corrected DNS to the VLAN gateway (`10.0.50.1`)**, both live and persistently via `pct set 2200 --nameserver 10.0.50.1`.

### Firewall Posture (pre-existing on DMZ interface, confirmed sound)
- Allow DMZ → gateway
- Block DMZ → all internal RFC1918 networks
- Allow DMZ → internet (outbound only)

### New Rule Added
`LAN net` → `10.0.50.69`, TCP port `81` only — permits ongoing admin access to the NPM web UI from inside the network. No rule permits inbound access to this DMZ from the internet; all public traffic arrives exclusively via the Cloudflare Tunnel.

---

## 2. Software Stack

Installed inside `npm-dmz`, in order:

1. **Docker** (official convenience script: `curl -fsSL https://get.docker.com | sh`)
2. **cloudflared** — current correct repo per Cloudflare docs as of Sept 2026:
   ```bash
   mkdir -p --mode=0755 /usr/share/keyrings
   curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
   echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | tee /etc/apt/sources.list.d/cloudflared.list
   apt-get update && apt-get install cloudflared
   ```
   `any` as the release codename works across Debian versions including Trixie for the `cloudflared` package specifically (note: a *different* Cloudflare product, `gokeyless`, requires the Trixie-specific repo path — not relevant here).
3. **Nginx Proxy Manager** and **ntfy** — via a single `docker-compose.yml` (below).

---

## 3. Cloudflare Tunnel Setup

### Prerequisite: Domain on Cloudflare
Domain `dcollinshomelab.org` (registered at Porkbun) added to Cloudflare on the Free plan. Nameservers at Porkbun updated to Cloudflare's assigned pair. **Note:** Porkbun's default parking-page DNS records (2× A, `*` and `www` CNAME to `pixie.porkbun.com`) had to be deleted before `cloudflared` could create its own CNAME routes — they'll conflict otherwise.

### Authentication
```bash
cloudflared tunnel login
```
Opens a URL for browser-based authorization against the target zone. Requires the zone to show **Active** in Cloudflare (post-nameserver-propagation) before it will appear in the zone picker.

### Tunnel Creation
```bash
cloudflared tunnel create web-tunnel
```
Tunnel ID: `85a7c629-3052-4d9c-8c4f-65c35406d680`. Credentials written to `/root/.cloudflared/<tunnel-id>.json`.

### Config File
`/root/.cloudflared/config.yml` (copied to `/etc/cloudflared/config.yml` automatically by service install):
```yaml
tunnel: 85a7c629-3052-4d9c-8c4f-65c35406d680
credentials-file: /root/.cloudflared/85a7c629-3052-4d9c-8c4f-65c35406d680.json

ingress:
  - hostname: dcollinshomelab.org
    service: http://localhost:80
  - hostname: '*.dcollinshomelab.org'
    service: http://localhost:80
  - service: http_status:404
```
Root domain and all subdomains route to NPM on port 80 (same container), which then does Host-header-based routing to the correct backend. The trailing `http_status:404` catch-all is required by `cloudflared`.

### DNS Routes
```bash
cloudflared tunnel route dns web-tunnel dcollinshomelab.org
cloudflared tunnel route dns web-tunnel '*.dcollinshomelab.org'
```
Creates CNAME records pointing at `<tunnel-id>.cfargotunnel.com`.

### Persistence
```bash
cloudflared service install
systemctl enable cloudflared
systemctl start cloudflared
```
Verified `active (running)`, all 4 edge connections registered (dfw01/06/08/13), full connectivity pre-check pass.

### Security Note
No WAN-side port forward exists or is required anywhere in this setup — `cloudflared` initiates an outbound-only connection to Cloudflare's edge. OPNSense's WAN interface remains fully default-deny with zero forwarding rules, consistent with the network's existing no-port-forwarding design principle. Standard Cloudflare guidance to "only allow Cloudflare IPs at your origin" does not apply here — there is no listening origin port to allowlist against; the tunnel's outbound-only model makes that class of exposure structurally impossible rather than something to configure.

---

## 4. docker-compose Stack

`/opt/npm-dmz/docker-compose.yml`:
```yaml
services:
  npm-dmz:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm-dmz
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  ntfy:
    image: 'binwiederhier/ntfy'
    container_name: ntfy
    command: serve
    restart: unless-stopped
    volumes:
      - ./ntfy/server.yml:/etc/ntfy/server.yml
      - ./ntfy/cache:/var/cache/ntfy
      - ./ntfy/data:/var/lib/ntfy
    ports:
      - '2586:80'
```

`/opt/npm-dmz/ntfy/server.yml`:
```yaml
base-url: "https://ntfy.dcollinshomelab.org"
upstream-base-url: "https://ntfy.sh"
listen-http: ":80"
cache-file: "/var/cache/ntfy/cache.db"
auth-file: "/var/lib/ntfy/auth.db"
auth-default-access: "deny-all"
behind-proxy: true
```

---

## 5. ntfy — Why It Needs the Tunnel, Not Just WireGuard

iOS background push delivery requires Apple's push infrastructure, whose certificates belong to the published ntfy app, not a self-hosted server. Self-hosted instances work around this via `upstream-base-url: https://ntfy.sh`: the self-hosted server sends a content-free "something arrived" signal to `ntfy.sh`, which relays a wake-up through Apple's actual push service; the phone then fetches the real message body directly from the self-hosted `base-url`.

**This means the phone must be able to reach the self-hosted server at the exact moment a notification fires.** Gating ntfy behind WireGuard-only access would mean notifications silently fail to complete any time the phone isn't connected to the VPN — defeating the purpose of an always-on alert channel. Hence: ntfy is deployed behind the public tunnel, not WireGuard-only.

### Placement: DMZ, Not services-host
Deploying inside `npm-dmz` (rather than `services-host`, where the other self-hosted apps live) avoids opening any new DMZ→internal firewall exception. Internal services/scripts that need to *publish* an alert (Uptime Kuma, a cron health check, etc.) simply `curl` the public `https://ntfy.dcollinshomelab.org/<topic>` endpoint like any external client — no special internal routing required, and the DMZ boundary stays exactly as clean as the rest of the network's isolation model.

### Access Control
`auth-default-access: deny-all` — closes the risk inherent to a public ntfy instance where topic names function as a de facto shared secret. User `dcollins` created with `admin` role (full read-write to all topics; simpler than per-topic ACLs for a single-user deployment). Anonymous access fully denied.

```bash
docker exec -it ntfy ntfy user add --role=admin dcollins
```

### NPM Proxy Host Configuration
| Field | Value |
|---|---|
| Domain | `ntfy.dcollinshomelab.org` |
| Scheme | `http` |
| Forward Hostname | `ntfy` (**Docker Compose service name — not `localhost`**) |
| Forward Port | `80` (ntfy's internal container port — not the host-published `2586`) |
| WebSockets Support | **On** (required for instant delivery; without it, falls back to slow polling) |
| SSL | Let's Encrypt, Force SSL on, Block Common Exploits on |

**Common mistake to avoid:** NPM and ntfy are separate containers on the same Docker Compose network. `localhost` inside NPM's container refers to NPM's own container, not the host or its sibling container — this produces a `502 Bad Gateway`. Use the Compose service name (`ntfy`) and the container's *internal* port (`80`), not the host-published port mapping.

### Verification
```bash
curl -u dcollins:<password> -d "test message" https://ntfy.dcollinshomelab.org/homelab-alerts
```
Confirmed end-to-end delivery to the ntfy iOS app (server added as `https://ntfy.dcollinshomelab.org`, logged in as `dcollins`, subscribed to `homelab-alerts`) — live over cellular data, no WireGuard connection required.

---

## Open Items

- **SSH Bastion + Cloudflare Access (admin tunnel)** — not yet built. Will gate SSH behind identity verification (Cloudflare Zero Trust Access, free up to 50 users) as a second, separately-scoped tunnel from the web tunnel documented here.
- **Existing (non-DMZ) NPM instance on services-host** — currently serves both public and internal/admin hostnames from one instance. Longer-term plan: internal-only, with all genuinely public traffic routed exclusively through the DMZ instance documented here.
- **Wire ntfy into alert sources** — Uptime Kuma has native ntfy notification support; disk-usage/health-check cron scripts on other nodes can `curl` the public endpoint directly, same as the manual test above.
- **kalshi-mm removal** — planned; will leave `pve-env1` with no other workloads sharing the node with `npm-dmz`.

# Runbook: Publish an Internal Service via a Dedicated Cloudflare Tunnel

**Category:** Cloudflare Tunnel
**When to use:** Making an internally-hosted service (running on a private CA-issued cert) reachable from the public internet without opening inbound firewall ports, and without adding it to an existing tunnel that fronts unrelated services.

## Why a dedicated tunnel/host

A single `cloudflared` instance and tunnel can carry multiple public hostname routes. Use a **separate** tunnel and host specifically when the service being published needs a different trust/isolation boundary than what's already flowing through an existing tunnel — e.g. publishing an identity provider should not share infrastructure with a tunnel that fronts a public portfolio site.

## Steps

1. **Build a minimal, single-purpose LXC** on the appropriate DMZ-tier VLAN (unprivileged, Debian, no nesting needed for `cloudflared` — it runs natively, no Docker required).

2. **Scope firewall rules narrowly** before installing anything: DNS/NTP to the VLAN's own gateway, and outbound to Cloudflare's published edge IPv4 ranges (`cloudflare.com/ips-v4`) on 443 — this single range covers both the tunnel connection itself and package installs pulled through Cloudflare's own CDN.

3. **Create the tunnel** in the Zero Trust dashboard (Networking → Tunnels → Create a tunnel → Cloudflared), name it distinctly from any existing tunnel.

4. **Install and register `cloudflared`** on the LXC using the install command the dashboard provides (copy the token fresh from the dashboard each time — do not retype or relay it through chat/notes, base64 tokens are extremely sensitive to a single dropped or substituted character).
   ```bash
   curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
   echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | tee /etc/apt/sources.list.d/cloudflared.list
   apt update && apt install cloudflared -y
   cloudflared service install <token>
   ```
   Confirm in the dashboard that the Connector shows **Connected**.

5. **Add the public hostname route** under the tunnel's **Published application routes** tab (not "Hostname routes" — that tab is for WARP-only private network routing and will not be reachable by external callers like an identity provider's backend).
   - Service Type: HTTPS (or HTTP, matching the origin)
   - URL: the origin's actual listening address

6. **Origin cert trust — check what's actually terminating TLS at the origin.** If the origin service uses a self-signed cert (many apps default to this on their own direct port) rather than one issued by your internal CA, either:
   - Route to a reverse proxy in front of it that *does* terminate TLS with a properly-issued cert (preferred — reuses existing trust infrastructure), or
   - Configure the tunnel's CA Pool setting for this specific route.

   To use a real internal CA:
   - Copy the CA root cert onto the tunnel LXC (via the Proxmox host if a direct network path doesn't exist, same as the step-ca runbook).
   - In the route's **Additional application settings → TLS**:
     - **Certificate Authority Pool**: local path to the root cert (e.g. `/etc/ssl/certs/homelab-root-ca.crt`)
     - **Origin Server Name**: the hostname matching the origin cert's CN/SAN
     - Leave **No TLS Verify** off — this setting name is deceptively similar to "Origin Server Name," don't confuse the two

7. **Verify from a genuinely external network** (cellular data, not the LAN — internal DNS overrides can make a broken public path look like it's working when tested from inside the network):
   ```
   curl -v https://<published-hostname>/<a known-good endpoint>
   ```

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Works from LAN, fails from outside | The hostname only ever had an internal DNS override; never actually routed through a tunnel | Confirm the tunnel's Routes/Published application routes column isn't empty |
| `Bad Gateway` at Cloudflare's edge | Tunnel reached the origin, but TLS handshake failed (self-signed cert, wrong CA Pool) | Step 6 above — check what's actually terminating TLS at the target IP:port |
| Wrong tab used for hostname route | "Hostname routes" (private network) picked instead of "Published application routes" | Only Published application routes are reachable without the requester running WARP |

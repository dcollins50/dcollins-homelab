---
name: cloudflare-tunnel-publish
type: runbook
tool: cloudflare
---

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

## Checking tunnel status

Menu names were checked against Cloudflare's documentation on Sep 24, 2026 and change often.

1. In the Cloudflare dashboard go to Networking, then Tunnels. Each tunnel has a status. Healthy means it is running and can carry traffic. Degraded means it can still carry traffic but something is wrong. Down means it has no connection to Cloudflare and cannot carry traffic. Inactive means it has never been run. The tunnels here are `web-tunnel` (on [[npm-dmz]]) and `auth-tunnel` (on [[pve-authtunnel]]).
2. Healthy only means the connector program on the container is connected to Cloudflare. It does not prove the service behind it works. On Sep 21, 2026 the auth tunnel was connected while visitors got Bad Gateway. Always finish by opening the public hostname from a phone on cellular data, not from the home network.
3. If a tunnel shows Down or Inactive, run `systemctl status cloudflared` on the container that runs it. It should say active (running).
4. The Connector logs and Routes columns in the tunnel list can be empty even when a tunnel works. On Sep 24, 2026 both tunnels showed Healthy with an empty Routes column, and `web-tunnel` had no connector logs link. `web-tunnel` was created from the command line with a local `config.yml` (see [[sop-cloudflare-tunnel-implementation]]), and the dashboard confirmed on Sep 24, 2026 that it is locally managed: hovering over the info icon next to its type says so and offers a Configure button to migrate it to the dashboard. That explains the empty columns. For a locally managed tunnel, the hostnames it serves are in the config file on its container, not in the dashboard. `auth-tunnel` was set up with a dashboard token, so it is dashboard-managed and has a View logs link.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Works from LAN, fails from outside | The hostname only ever had an internal DNS override; never actually routed through a tunnel | Open the tunnel in the dashboard and check that it has a route for the hostname (or check the config file on the container for a file-managed tunnel). Do not rely on the Routes column in the tunnel list, which was empty for working tunnels on Sep 24, 2026 |
| `Bad Gateway` at Cloudflare's edge | Tunnel reached the origin, but TLS handshake failed (self-signed cert, wrong CA Pool) | Step 6 above — check what's actually terminating TLS at the target IP:port |
| Wrong tab used for hostname route | "Hostname routes" (private network) picked instead of "Published application routes" | Only Published application routes are reachable without the requester running WARP |

## Related Documentation

- [[npm-dmz]]
- [[VLAN51]]
- [[cloudflare-warp-zero-trust-enrollment|Cloudflare WARP Enrollment]]

# ntfy / Authentik Outpost Troubleshooting — September 15, 2026

## Trigger

ntfy alerts stopped working after unsubscribing from a topic to test reauth. Resubscribing failed from both the browser web app and the phone app, with the browser showing "error sending alert."

## Timeline of root causes

This turned into a chain of separate bugs, each one uncovered only after fixing the one before it. Full sequence, in order:

### 1. authentik-proxy container unhealthy (DNS)

`docker ps` on npm-dmz showed the `authentik-proxy` outpost container as unhealthy. Its logs showed a DNS resolution failure trying to reach `authentik.dcollinshomelab.org` to pull config from the Authentik API.

**Cause:** `authentik.dcollinshomelab.org` had no internal DNS override. npm-dmz's resolver (OPNSense's Unbound at 10.0.50.1) was forwarding the query to public DNS and getting NXDOMAIN, since that hostname was deliberately never given a public Cloudflare record.

**Fix:** Added an Unbound host override (Services > Unbound DNS > Overrides) mapping `authentik.dcollinshomelab.org` → `10.0.30.10` (the Authentik LXC's IP at the time).

### 2. DMZ → Dev VLAN firewall block

After the DNS fix, the outpost's error changed from DNS failure to `ConnectError ... TimedOut` on `10.0.30.10:443`.

**Cause:** DMZ's default firewall posture blocks DMZ → all internal RFC1918 networks. There was no explicit allow rule for npm-dmz to reach the Authentik host on Dev VLAN (VLAN30).

**Fix:** Added a Pass rule on the DMZ interface, source `npm_dmz` alias (10.0.50.69), destination `Authentik` alias, placed above the existing "Block DMZ to all internal VLANs" rule (DMZ rules evaluate first-match). Initially scoped to port 443.

### 3. Wrong destination port (connection refused)

With the firewall opened, the error changed again to `ConnectionRefused` on port 443.

**Cause:** Authentik's server container doesn't listen on 443 internally. `ss -tlnp` on the Authentik LXC confirmed it listens on 9000 (HTTP) and 9443 (HTTPS).

**Fix:** Updated the DMZ firewall rule's destination port from 443 to 9443. Updated `AUTHENTIK_HOST` and `AUTHENTIK_HOST_BROWSER` in npm-dmz's `docker-compose.yml` to include `:9443`. Recreated the container with `docker compose up -d authentik-proxy`. Outpost went healthy, connected to the Authentik websocket, and began serving the `ntfy-web` application.

### 4. Outpost intercepting token-authenticated requests

With the outpost healthy, ntfy publish requests (using Basic auth) returned a `302` redirecting into the Authentik login flow instead of reaching ntfy.

**Cause:** The `auth_request` directive in NPM's custom nginx config for `ntfy.dcollinshomelab.org` ran unconditionally on every request, regardless of whether the request already carried its own auth (Basic/Bearer header from ntfy's own token auth, e.g. Kuma's publisher account or the phone app).

**Fix:** Added an internal `/internal-auth-check` location that returns `200` immediately when `$http_authorization` matches `^(Basic|Bearer)\s`, bypassing the Authentik check entirely for token-authenticated requests. Only requests without that header (a browser hitting the UI cold) fall through to the real Authentik auth_request.

### 5. Missing proxy_pass to ntfy backend

Fixing #4 exposed a new error: authenticated requests now returned a clean `404` instead of a redirect.

**Cause:** The `location /` block in NPM's custom config had the Authentik auth wiring but no `proxy_pass` to the ntfy container at all. It had likely never actually forwarded traffic to ntfy; the earlier unconditional redirect had been masking this the whole time.

**Fix:** Added `proxy_pass http://ntfy:80;` plus standard proxy headers, `proxy_http_version 1.1`, and `Upgrade`/`Connection: upgrade` headers (needed for ntfy's live SSE-style subscriptions) into the `location /` block.

**Result:** `curl -u dcollins:*** https://ntfy.dcollinshomelab.org/kuma-alerts -d "test message"` returned a clean `200` with a real ntfy message ID. Chain confirmed working end to end.

### 6. Web app not live-updating (proxy buffering)

Phone push notifications arrived instantly, but the ntfy browser web app never showed new messages live, and topic loading spinners never resolved.

**Cause:** nginx buffers proxied responses by default, holding the SSE stream's data instead of forwarding it live as it arrives.

**Fix:** Added `proxy_buffering off;`, `proxy_cache off;`, `chunked_transfer_encoding off;`, and `proxy_read_timeout 1h;` to the ntfy `location /` block.

### 7. Authentik browser redirect broken after the port fix

Fixing #3 (adding `:9443` to `AUTHENTIK_HOST_BROWSER`) broke browser access to Authentik's own login page: cert warnings, then a clean `404` once the self-signed cert was accepted.

**Cause:** `AUTHENTIK_HOST` (server-to-server, outpost talking to Authentik) and `AUTHENTIK_HOST_BROWSER` (what the browser gets redirected to) serve different purposes and shouldn't both carry the internal port. Authentik's own admin-configured Base URL is `https://authentik.dcollinshomelab.org` with no port at all, and Authentik enforces that its own frontend is only reachable at that exact address. More fundamentally, `authentik.dcollinshomelab.org` had never been given a real reverse proxy; it was resolving straight to the raw Authentik LXC on its raw container ports, which is why any URL with a port kept breaking.

**Fix:**
- Set up a proper NPM proxy host for `authentik.dcollinshomelab.org` on the **internal** NPM instance (10.0.20.30, services-host), not npm-dmz, forwarding to `10.0.30.10:9000` with a real internal PKI (Homelab Intermediate CA) certificate. This matches the pattern used for other internal-only `homelab.local`-style services and keeps Authentik off the public-facing DMZ NPM instance entirely.
- Repointed the Unbound DNS override for `authentik.dcollinshomelab.org` from `10.0.30.10` to `10.0.20.30` (the internal NPM, not the raw LXC).
- Reverted `AUTHENTIK_HOST_BROWSER` back to `https://authentik.dcollinshomelab.org`, no port. Left `AUTHENTIK_HOST` at `:9443` (server-to-server, correctly bypasses the proxy).
- Added a new alias `internal_npm` (10.0.20.30) and a Pass rule on the Services (VLAN20) interface, source `internal_npm`, destination `Authentik` alias, port 9000, placed above the Services→internal-VLANs block rule.

### 8. Stray Pi-hole AAAA record

Even after the internal proxy was live, the browser still failed to connect cleanly.

**Cause:** `nslookup authentik.dcollinshomelab.org` from the workstation (using Pi-hole, not Unbound, as the client resolver) returned both the correct `10.0.20.30` A record and a stray `::1` AAAA record. Browsers prefer IPv6 when both are present, so the browser was trying to connect to itself (loopback) instead of the internal NPM proxy. This is the same class of split-horizon bug documented from the original Authentik DEV VLAN rework session.

**Fix:** Removed the stray `::1` local DNS record from Pi-hole, leaving only the correct A record.

### 9. False alarm: stale browser security state

After all of the above, Chrome's Security tab still showed "not secure / active content with certificate errors" on the Authentik login page, even though all network requests (including the websocket handshake, confirmed via a `101 Switching Protocols` response) were succeeding cleanly.

**Cause:** Stale per-tab security state cached from earlier in the session, before the real fixes landed. Confirmed via a private/incognito window, which loaded the login page cleanly with "Connection is secure."

**Fix:** None needed beyond a full browser restart to clear the cached state in the normal window.

## End state

- ntfy alerts work end to end: phone push, browser live updates, and Kuma's automated publish calls all functioning.
- Authentik now has a proper internal-only reverse proxy through the internal NPM instance (10.0.20.30), consistent with the DNS/PKI pattern used for other internal `homelab.local`-style services, rather than resolving directly to the raw LXC.
- New firewall paths opened and documented: DMZ (npm-dmz) → Dev (Authentik) on 9443, and Services (internal NPM) → Dev (Authentik) on 9000.
- New aliases created: `npm_dmz` (10.0.50.69), `internal_npm` (10.0.20.30). Existing `Authentik` alias reused for both new rules.

## Open items / things worth double-checking later

- Confirm the DMZ → Authentik (9443) firewall path is actually still needed long-term, or whether it should be narrowed further now that the outpost's primary config-fetch traffic is the main thing using it.
- Consider whether Pi-hole should be set to conditionally forward `dcollinshomelab.org` queries to Unbound entirely, rather than holding its own separate local records, to prevent this same split-horizon class of bug from recurring on other internal hostnames.
- No formal confirmation yet that the Wazuh-to-ntfy integration item (still on the Phase 3 backlog) will interact cleanly with the new auth-bypass-on-token-header nginx logic; worth a quick sanity check once that integration is actually built.

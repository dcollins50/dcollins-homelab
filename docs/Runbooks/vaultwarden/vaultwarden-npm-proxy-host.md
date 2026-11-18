---
name: vaultwarden-npm-proxy-host
type: runbook
tool: vaultwarden
---

Put Vaultwarden behind the internal Nginx Proxy Manager on [[services-host]] with TLS and working websockets. General steps for any service are in [[npm-add-proxy-host]]. This note only holds the Vaultwarden specific parts. The address Vaultwarden is actually served on is <FILL IN: hostname>.

## Why it matters
Vaultwarden needs HTTPS to work properly (browsers refuse the web crypto API on plain HTTP, and the admin docs call for HTTPS). It also pushes sync notifications over websockets, so the proxy has to pass the upgrade headers.

## Steps (internal NPM)
1. NPM, Hosts, Proxy Hosts, Add Proxy Host.
2. Domain: <FILL IN: vault hostname>. Forward scheme: `http`. A 502 earlier in this homelab came from the scheme being set to `https` for a plain HTTP backend, see [[npm-troubleshoot-proxy-routing]].
3. Forward host and port. Two possible shapes, pick the one that matches the real setup:
   - Container name and port `80`, if NPM and Vaultwarden share the Docker network `reverse-proxy-net`.
   - `10.0.20.x` and port `8080`, using the published port.
4. Turn on **Websockets Support** and Block Common Exploits.
5. SSL tab: certificate from the internal CA, per [[npm-configure-tls-cert]]. Force SSL on. The planned step-ca certificate is an open item in [[vaultwarden-integration]].
6. In Vaultwarden set `DOMAIN` to the exact `https://` address, so attachments, invitations and WebAuthn links are built correctly.
7. Make sure the real client IP reaches Vaultwarden (`X-Real-IP` and `X-Forwarded-For`). Without it every login looks like it comes from the proxy, which breaks [[vaultwarden-configure-fail2ban]].

## Verify
- `https://<vault hostname>/api/config` returns JSON.
- Edit an item in one client and watch it appear in another without a manual sync (confirms websockets).

## If the page loads but sync is slow
Check the websocket request in the browser DevTools Network tab, filter WS, and read its status. 101 means the upgrade worked.

## Related
- [[npm-add-proxy-host]]
- [[npm-configure-tls-cert]]
- [[npm-fix-502-upstream-header-too-big]]
- [[vaultwarden]]

Note: the official Vaultwarden proxy page has generic nginx examples (upstream with keepalive, `Upgrade` and `Connection` headers, `X-Real-IP`) but no NPM specific steps. The NPM steps above come from this homelab's own sessions.

Source: https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples

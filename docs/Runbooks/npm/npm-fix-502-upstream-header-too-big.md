---
name: npm-fix-502-upstream-header-too-big
type: runbook
tool: npm
---

# Runbook: Fix 502 "upstream sent too big header" in Nginx Proxy Manager

**When to use:** A proxied app works for normal pages but returns a 502 on one specific request, usually a login or OIDC redirect that carries a very large URL or `Set-Cookie` header. Found Oct 2, 2026 on the `authentik` proxy host in [[services-host]], where the Authentik authorize redirect for Cloudflare WARP enrollment failed.

## Symptoms
- Browser shows `ERR_QUIC_PROTOCOL_ERROR`, or a Cloudflare 502 page, on the redirect only. Chrome QUIC was already disabled, so QUIC was a red herring.
- When the app is published through a Cloudflare Tunnel, the 502 is Cloudflare's page, and `cloudflared` logs nothing useful. The real error is in NPM.

## Diagnose
1. On services-host, find the NPM container logs (`nginx-proxy-manager`, logs under `/data/logs/` inside the container).
2. Look in the proxy host's error log for `upstream sent too big header while reading response header from upstream`. That line confirms this runbook applies.

## Fix
NPM, Proxy Hosts, edit the host, Advanced tab, Custom Nginx Configuration:
```
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```
Save. Retry the failing request.

## Notes
- The Sep 21 `ERR_QUIC_PROTOCOL_ERROR` was never proven to be this same cause. It is the most likely one, since enrollment completed right after this change.
- Do not apply this to every proxy host. Add it only where the log line above appears.

## Related Documentation
- [[npm-troubleshoot-proxy-routing|Troubleshoot NPM Routing]]
- [[npm-add-proxy-host|Add a Proxy Host]]
- [[cloudflare-warp-zero-trust-enrollment]]

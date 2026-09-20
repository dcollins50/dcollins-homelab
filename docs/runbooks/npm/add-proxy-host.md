# Runbook: Add a Proxy Host (Nginx Proxy Manager)

| Field | Value |
| --- | --- |
| Applies to | Nginx Proxy Manager |
| Category | Reverse Proxy |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new service needs to be reachable through a friendly hostname with TLS termination, rather than users hitting an internal IP and port directly. This environment runs **two separate NPM instances**, decide which one applies before starting:

- **Internal NPM** (services-host, VLAN20) — for internal-only services, resolved via `*.homelab.local`
- **npm-dmz** (VLAN50) — dedicated to public-facing services reached through the Cloudflare Tunnel, kept separate from the internal instance deliberately, don't mix internal and public-facing proxy hosts on the same instance

---

## Before you start

- **Confirm which NPM instance this belongs on**, per the split above.
- **Know the backend's actual address and port.** The proxy host forwards to this, not to a hostname that might itself resolve back through a proxy (see the routing note below).
- **Decide the TLS approach:**
  - Internal NPM: certs issued from this environment's internal PKI (Intermediate CA), see [Configure TLS Certificates](configure-tls-cert.md)
  - npm-dmz: typically Let's Encrypt (NPM's built-in ACME support) since it's public-facing, or a custom cert if there's a specific reason not to use Let's Encrypt

---

## Steps

1. **Log into the correct NPM instance's admin UI** (port 81).
2. **Hosts → Proxy Hosts → Add Proxy Host.**
3. **Details tab:**
   - Domain Names: the hostname(s) this proxy host answers for
   - Scheme: http or https, matching what the backend actually serves
   - Forward Hostname/IP: **the backend's direct IP**, not a hostname that resolves through another proxy layer, avoids the routing trap covered in [Troubleshoot Proxy Routing](troubleshoot-proxy-routing.md)
   - Forward Port: the backend's actual listening port
   - Cache Assets / Block Common Exploits / Websockets Support: enable as appropriate for the specific service (Websockets Support matters for anything using long-lived connections, e.g. real-time dashboards)
4. **SSL tab:**
   - Request a new certificate (internal PKI cert for internal NPM, Let's Encrypt for npm-dmz) or select an existing one
   - Force SSL: enable, this environment's services are consistently HTTPS-only through NPM
   - HTTP/2 Support: enable unless the backend has a specific reason not to support it
5. **Save.**
6. **Add the corresponding DNS record** if one doesn't already exist:
   - Internal NPM: a Pi-hole Local DNS Record pointing the hostname at the NPM instance's IP, see [Add a Local DNS Record](../pihole/add-local-dns-record.md)
   - npm-dmz: handled via the Cloudflare Tunnel configuration, not Pi-hole
7. **Test** by visiting the hostname, confirm it resolves, terminates TLS correctly (no cert warnings), and actually reaches the backend service.

---

## Common mistakes

- **Adding a proxy host on the wrong instance** (e.g. an internal-only service accidentally added to npm-dmz), exposing something that was never meant to be public-facing.
- **Forwarding to a hostname instead of a direct IP**, especially one that itself resolves through another proxy, causes confusing routing failures, see [Troubleshoot Proxy Routing](troubleshoot-proxy-routing.md).
- **Forgetting the DNS record**, the proxy host is configured correctly but nothing resolves to it yet.
- **Not enabling Websockets Support** for a service that needs it, resulting in a service that loads but behaves oddly (dashboards that don't live-update, etc.).

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Configure TLS Certificates](configure-tls-cert.md)
- [Troubleshoot Proxy Routing](troubleshoot-proxy-routing.md)
- [Add a Local DNS Record](../pihole/add-local-dns-record.md)

# Runbook: Troubleshoot Proxy Routing (Nginx Proxy Manager)

| Field | Value |
| --- | --- |
| Applies to | Nginx Proxy Manager |
| Category | Troubleshooting |
| Author | Daniel Collins |

---

## When to use this

Use this when a service reachable directly by IP doesn't work through its NPM hostname, or when a backend-to-backend connection mysteriously times out despite everything looking correctly configured. This runbook exists because of a real, already-hit failure mode in this environment, not a hypothetical.

---

## The core issue: NPM only proxies 80/443

**Nginx Proxy Manager's proxy hosts forward standard web traffic (HTTP/HTTPS). They do not forward arbitrary non-standard ports.** If a backend service needs to be reached on a different port (e.g. Elasticsearch on 9200), pointing that connection at the NPM hostname will not work, NPM has no proxy host rule for that port and the connection times out or fails.

**This already happened here:** Filebeat was initially pointed at `elasticsearch.homelab.local` (which resolves to the internal NPM IP) for its connection to Elasticsearch on port 9200. NPM only proxies 80/443, so the connection failed. The fix was to point Filebeat directly at Elasticsearch's IP on port 9200, bypassing NPM entirely for that backend connection. See [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md) for how the certificate's IP SAN supports this.

**The general rule:** NPM (or any reverse proxy) is for browser-facing web traffic. Backend service-to-service connections on non-standard ports should connect directly to the service's IP, never routed through a reverse proxy that only handles 80/443.

---

## Diagnostic steps

1. **Is the failing connection actually going through NPM, or should it be bypassing it?**
   - If it's a browser hitting a hostname expecting a normal web page: NPM should be involved, check the proxy host config.
   - If it's a backend service connecting on a specific non-standard port: NPM should almost certainly **not** be involved, check whether the config is mistakenly pointed at the NPM hostname instead of the backend's direct IP.

2. **If it's a legitimate browser/web request through NPM that's failing:**
   - Confirm the proxy host exists and the domain matches exactly (including subdomain).
   - Confirm the Forward Hostname/IP and Forward Port in the proxy host config actually match the backend's current address, a backend that moved IP without the proxy host being updated is a common, easy-to-overlook cause.
   - Check NPM's own logs for the specific host (**Proxy Hosts → [host] → Edit**, or the container/service logs directly) for the actual error, rather than guessing.

3. **If it's a backend-to-backend connection that should bypass NPM:**
   - Confirm the service's configuration actually points at the backend's direct IP and port, not a `*.homelab.local` hostname.
   - If it currently does point at a hostname and needs to keep doing so for some reason, confirm that hostname resolves to the actual backend directly (via a dedicated internal DNS record) rather than to the NPM IP.

4. **DNS resolution vs. routing, distinguish these:**
   ```bash
   dig <hostname>
   ```
   Confirms what IP the hostname actually resolves to. If it resolves to the NPM IP but the connection needs a non-standard port, that's the mismatch, the DNS is "working" but pointed at the wrong layer for this kind of traffic.

---

## Common mistakes

- **Assuming a working hostname means all traffic to that service should go through it.** A hostname resolving to NPM is fine for browser/web traffic and actively wrong for backend connections needing a non-proxied port.
- **Debugging TLS or firewall issues first** when the actual problem is architectural (wrong layer entirely), check whether NPM should even be in the path before troubleshooting deeper.
- **Fixing the immediate symptom (pointing one service at the direct IP) without checking for other configs making the same mistake.** If one backend connection was wired through NPM incorrectly, check whether others might be too.

---

## Related Documentation

- [Add a Proxy Host](add-proxy-host.md)
- [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md)
- [Network Architecture](../../network.md)

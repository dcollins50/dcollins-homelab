# Runbook: Configure TLS Certificates (Nginx Proxy Manager)

| Field | Value |
| --- | --- |
| Applies to | Nginx Proxy Manager |
| Category | Security / PKI Integration |
| Author | Daniel Collins |

---

## When to use this

Use this when attaching a certificate to a new proxy host, renewing/reissuing one, or troubleshooting a TLS warning on a proxied service. The right approach differs by which NPM instance is involved, see below.

---

## Internal NPM (VLAN20) — internal PKI certs

Internal services use certificates issued by this environment's Intermediate CA, not Let's Encrypt (internal `*.homelab.local` hostnames aren't publicly resolvable, so ACME validation wouldn't work for them anyway).

1. **Issue the certificate from the Intermediate CA** for the hostname in question, see [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md) for the actual `openssl` issuance steps, the process is the same regardless of which service the cert is for.
2. **In NPM: SSL → Certificates → Add SSL Certificate → Custom.**
3. Upload the cert and key.
4. **Also upload the CA chain** (intermediate + root) if NPM's custom cert flow supports a chain/intermediate field, so clients trusting the internal Root CA validate cleanly.
5. Attach the certificate to the proxy host (**Proxy Hosts → [host] → Edit → SSL tab → select the certificate**).
6. Force SSL, enable.

### Renewing/reissuing an internal cert

Same process as issuing new, but **revoke the old certificate on the Intermediate CA first** if reusing the same CN, see the mismatch warning in [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md), the same CA behavior applies here.

---

## npm-dmz (VLAN50) — public-facing certs

Public-facing services can use NPM's built-in Let's Encrypt support directly, since these hostnames are actually publicly resolvable.

1. **Proxy Hosts → [host] → Edit → SSL tab.**
2. **Request a new SSL Certificate**, select Let's Encrypt.
3. Provide a valid email for renewal notices.
4. **I Agree to the Let's Encrypt Terms of Service**, enable.
5. Save, NPM handles domain validation and issuance automatically (requires the domain to actually be reachable from the internet for HTTP-01 validation, or DNS-01 if configured with a DNS provider plugin).
6. **NPM auto-renews Let's Encrypt certs**, no manual renewal step under normal operation, but worth confirming a renewal actually succeeded if a cert is ever found expired unexpectedly (check NPM's own logs).

### If Let's Encrypt issuance fails

Common causes: the domain isn't actually pointing at a reachable endpoint for HTTP-01 validation (check the Cloudflare Tunnel routing is live before requesting the cert, not after), or a rate limit was hit from repeated failed attempts, don't just retry immediately, check what actually failed first in NPM's request log.

---

## Common mistakes

- **Trying to use Let's Encrypt for an internal `*.homelab.local` hostname**, this will never succeed since it's not publicly resolvable, use the internal PKI path instead.
- **Uploading a custom cert without the full chain**, causes clients that don't already trust the intermediate directly to fail validation even though the cert itself is valid.
- **Reissuing an internal cert without revoking the old one first**, same mismatch risk documented in the ELK TLS runbook.
- **Requesting a public cert before the domain is actually reachable**, guarantees an avoidable failed validation attempt.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Add a Proxy Host](add-proxy-host.md)
- [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md)

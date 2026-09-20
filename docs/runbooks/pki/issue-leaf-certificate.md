# Runbook: Issue a Leaf Certificate (Internal PKI)

| Field | Value |
| --- | --- |
| Applies to | Internal PKI (Intermediate CA) |
| Category | Certificate Management |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a service needs a TLS certificate signed by this environment's internal CA, this is the base procedure that other runbooks (TLS on Elasticsearch/Kibana, NPM certs) reference rather than repeat. The Intermediate CA handles all day-to-day issuance, the Root CA is never involved in issuing leaf certs directly, see [Bring the Root CA Online](bring-root-ca-online.md) for when the Root CA actually needs to be touched.

---

## Before you start

- **Know every SAN the cert needs**, not just the primary hostname. If any backend service will connect by IP rather than hostname (common for service-to-service connections that go around a reverse proxy), include an IP SAN now, adding it after the fact means reissuing, this has already caused an avoidable outage in this environment once (see the Filebeat/NPM lesson in [Troubleshoot Proxy Routing](../npm/troubleshoot-proxy-routing.md)).
- **Check whether this CN already has a certificate.** If so, this is a reissue, not a fresh issuance, see [Revoke a Certificate](revoke-certificate.md) first, the Intermediate CA will not cleanly reissue for an existing CN without revocation.

---

## Steps

On the Intermediate CA VM:

1. **Generate a private key:**
   ```bash
   openssl genrsa -out <service>.key 4096
   ```

2. **Generate a certificate signing request:**
   ```bash
   openssl req -new -key <service>.key -out <service>.csr \
     -subj "/CN=<service>.homelab.local/O=Homelab"
   ```

3. **Sign the CSR with the Intermediate CA**, including every SAN identified up front:
   ```bash
   openssl x509 -req -in <service>.csr \
     -CA intermediate-ca.crt -CAkey intermediate-ca.key \
     -CAcreateserial -out <service>.crt \
     -days 365 -sha256 \
     -extfile <(printf "subjectAltName=DNS:<service>.homelab.local,IP:<service-ip>")
   ```
   Adjust the `-days` validity period only if there's a specific reason to deviate from the standard 365-day issuance used elsewhere in this environment.

4. **Verify the cert/key pair actually match** before deploying anywhere:
   ```bash
   openssl x509 -noout -modulus -in <service>.crt | openssl md5
   openssl rsa -noout -modulus -in <service>.key | openssl md5
   ```
   Both outputs must match. Don't skip this, a mismatched pair has made it past issuance undetected here before.

5. **Copy the cert, key, and CA chain** (intermediate + root cert) to the target host. The full chain is needed for any client to validate up to the trusted Root CA, not just the intermediate, see [Distribute the Trust Store](distribute-trust-store.md) if this is a client that doesn't already trust the Root CA.

6. **Deploy and restart the service** using the new cert.

7. **Validate the deployed cert:**
   ```bash
   openssl s_client -connect <host>:<port> -CAfile root-ca.crt
   ```
   Confirm the chain validates cleanly, then confirm the actual application works end to end, not just that the TLS handshake succeeds.

---

## Common mistakes

- **Missing a SAN that a backend connection actually needs**, most commonly an IP SAN for a service-to-service connection that bypasses a reverse proxy.
- **Reissuing without revoking the old cert first**, risks a mismatched cert/key pair, see [Revoke a Certificate](revoke-certificate.md).
- **Skipping the modulus verification step**, a fast check that catches a real, previously-encountered failure mode.
- **Deploying the leaf cert without the full CA chain**, breaks validation for any client that doesn't already trust the intermediate directly.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Revoke a Certificate](revoke-certificate.md)
- [Distribute the Trust Store](distribute-trust-store.md)
- [Configure TLS (Elasticsearch/Kibana)](../elk/configure-tls-elasticsearch-kibana.md)

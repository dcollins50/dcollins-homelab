# Runbook: Configure TLS (Elasticsearch/Kibana)

| Field | Value |
| --- | --- |
| Applies to | Elasticsearch, Kibana, Logstash, Filebeat |
| Category | Security / PKI Integration |
| Author | Daniel Collins |

---

## When to use this

Use this when standing up TLS for the first time on a new Elastic Stack component, reissuing a certificate (e.g. adding a SAN, renewing), or troubleshooting a TLS/chain-validation failure between stack components. This environment's internal PKI (Root CA → Intermediate CA) issues all of these certs, see [Internal PKI](../../pki.md) for the CA hierarchy itself.

---

## Before you start

- **Decide the SANs the cert needs.** At minimum the hostname it'll be reached by (`*.homelab.local`), and **an IP SAN if any backend service-to-service connection will use the IP directly rather than the hostname** (see the Filebeat lesson below, this one has actually caused an outage here before).
- **Know whether this is a fresh cert or a reissue.** The internal CA (raw OpenSSL currently, see [Internal PKI](../../pki.md) for the step-ca migration status) refuses to reissue a cert for an existing CN without first revoking the old one.

---

## Steps

### 1. Issue the certificate from the Intermediate CA

On the Intermediate CA VM:

```bash
openssl genrsa -out <service>.key 4096

openssl req -new -key <service>.key -out <service>.csr \
  -subj "/CN=<service>.homelab.local/O=Homelab"

openssl x509 -req -in <service>.csr \
  -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -out <service>.crt \
  -days 365 -sha256 \
  -extfile <(printf "subjectAltName=DNS:<service>.homelab.local,IP:<service-ip>")
```

If reissuing an existing CN, **revoke the old certificate first**, attempting to reissue without revoking can return the old cert paired with a newly generated key, a silent mismatch that only surfaces later as a confusing TLS failure.

### 2. Verify the cert/key pair actually match before deploying

```bash
openssl x509 -noout -modulus -in <service>.crt | openssl md5
openssl rsa -noout -modulus -in <service>.key | openssl md5
```

Both outputs must match. This step exists because of a real incident here where a mismatched pair made it past issuance undetected, verify every time, not just when something seems wrong.

### 3. Deploy to the service

1. Copy the cert, key, and the CA chain (intermediate + root) to the target host.
2. Configure the service to use them:
   - **Elasticsearch:** `xpack.security.http.ssl` (cert + key)
   - **Kibana:** `elasticsearch.ssl.certificateAuthorities` needs **both** the intermediate and root CA certs, the intermediate cert alone is not sufficient for Kibana to build a full trust chain
   - **Logstash:** `ssl_certificate_authorities` in the relevant output block(s), see [Add a Log Source / Pipeline](add-log-source-pipeline.md) if this is a new pipeline
3. Restart the service.

### 4. Point backend connections at the IP, not the hostname, where appropriate

If the hostname resolves through a reverse proxy (NPM) that only forwards standard web ports (80/443), any backend service connecting on a non-standard port (e.g. Elasticsearch's 9200) **must connect by IP**, the hostname will resolve but the proxy won't pass that port through, causing a timeout that looks like a TLS or network problem but is actually a routing one. This is exactly why the IP SAN in step 1 matters, add it up front for any cert that a backend service will need to reach directly.

### 5. Validate

```bash
openssl s_client -connect <host>:<port> -CAfile root-ca.crt
```

Confirm the chain validates cleanly. Then confirm the actual application-level connection works (Kibana loads without cert errors, Filebeat/Logstash ships without TLS errors in their logs), a chain that validates via `openssl s_client` but an app that still errors usually means the app's specific trust-store config is missing a piece (commonly, only the intermediate and not the root).

---

## Common mistakes

- **Reissuing without revoking first**, producing a mismatched cert/key pair.
- **Only including the intermediate CA cert in Kibana's `certificateAuthorities`**, missing the root breaks chain validation.
- **Missing the IP SAN**, then having a backend service fail to connect because it was pointed at a hostname that resolves through a proxy that doesn't pass the needed port.
- **Not verifying modulus match before deploying**, catching a mismatch after the fact costs more time than the 30 seconds this check takes.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [SOC Stack](../../soc-stack.md)
- [Add a Log Source / Pipeline](add-log-source-pipeline.md)

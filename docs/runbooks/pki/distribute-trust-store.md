# Runbook: Distribute the Trust Store (Internal PKI)

| Field | Value |
| --- | --- |
| Applies to | Internal PKI (Root CA distribution) |
| Category | Certificate Management |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new client, host, or browser needs to trust certificates issued by this environment's internal CA. Without the Root CA certificate in its trust store, a client will see every internally-issued certificate as untrusted, even though the cert itself is valid, this is a client-side trust problem, not a certificate problem, and is a different fix than reissuing anything.

---

## Steps

### Linux hosts (Debian/Ubuntu-based)

1. Copy the Root CA certificate to the host.
2. Install it into the system trust store:
   ```bash
   sudo cp root-ca.crt /usr/local/share/ca-certificates/homelab-root-ca.crt
   sudo update-ca-certificates
   ```
3. Verify:
   ```bash
   openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt <some-leaf-cert>.crt
   ```
   Should return `OK` for a cert issued by the internal chain.

### Browsers

1. Import the Root CA certificate into the browser's own certificate store, under Trusted Root Certificate Authorities (exact menu path varies by browser).
2. Restart the browser if it doesn't pick up the change immediately.
3. Visit an internally-issued HTTPS service and confirm no certificate warning appears.

### Other platforms (e.g. a different OS, a specific application with its own trust store)

The principle is the same regardless of platform: the Root CA cert needs to land in whatever trust store that specific client/OS/application actually consults, not necessarily the system-wide one. Some applications maintain their own separate trust store independent of the OS, check whether that's the case before assuming a system-level install covers it.

---

## Verifying trust is actually working

Don't just confirm the cert was added, confirm the actual effect:

1. From the client, connect to a service using an internally-issued certificate.
2. Confirm no trust warning/error appears, and that this is because the chain validates, not because the client is configured to skip validation entirely (that would hide a real problem rather than fix it).
3. If using `openssl s_client` for the test rather than a real client, use `-CAfile` pointed at the Root CA cert specifically, to confirm the chain validates against it deliberately, not just that some overly-permissive default happens to pass.

---

## Common mistakes

- **Installing the Intermediate CA cert instead of (or without) the Root CA cert** into a trust store, the Root is the actual trust anchor, a client needs to trust it directly, not just have a copy of the intermediate lying around.
- **Assuming a system-level trust store install covers every application on that host.** Some applications (certain language runtimes, specific software with bundled CA lists) maintain independent trust stores that need separate handling.
- **Confirming "no cert warning" without checking why.** A client configured to skip TLS validation entirely will also show no warning, that's not the same as trust actually being established correctly.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Issue a Leaf Certificate](issue-leaf-certificate.md)

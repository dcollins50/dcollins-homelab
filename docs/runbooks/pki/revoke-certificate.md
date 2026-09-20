# Runbook: Revoke a Certificate (Internal PKI)

| Field | Value |
| --- | --- |
| Applies to | Internal PKI (Intermediate CA) |
| Category | Certificate Management |
| Author | Daniel Collins |

---

## When to use this

Use this before reissuing a certificate for a CN that already has one (the Intermediate CA won't cleanly reissue over an existing CN without this step first), or when a certificate needs to be invalidated outright, a key suspected compromised, a service being decommissioned, a SAN that's no longer accurate.

---

## Before you start

- **Confirm the exact certificate being revoked** (CN, serial number), revoking the wrong certificate on a CA with multiple issued certs is not something to guess at.
- **Know why**: routine reissue (SAN change, renewal) versus an actual security concern changes the urgency but not the mechanics below.

---

## Steps

On the Intermediate CA VM:

1. **Identify the certificate's serial number**, from the cert file itself if you have it:
   ```bash
   openssl x509 -noout -serial -in <service>.crt
   ```
   or from the CA's own issued-certificate records if working from the CN alone.

2. **Revoke the certificate:**
   ```bash
   openssl ca -config <ca-config-path> -revoke <service>.crt \
     -cert intermediate-ca.crt -keyfile intermediate-ca.key
   ```
   Adjust the config path to match this environment's actual CA directory structure.

3. **Regenerate the CRL (Certificate Revocation List)** if this environment's clients/services actually check CRLs:
   ```bash
   openssl ca -config <ca-config-path> -gencrl \
     -cert intermediate-ca.crt -keyfile intermediate-ca.key \
     -out intermediate-ca.crl
   ```
   Note: this environment's current services largely validate certs via chain-of-trust rather than active CRL checking, so this step's practical effect may be limited today, it's included for completeness and in case that changes. Confirm whether this actually matters for a given service before assuming revocation alone takes it out of service.

4. **If revoking due to a suspected compromise** (not just a routine reissue), also:
   - Remove/replace the certificate on the affected service immediately, don't wait for a convenient maintenance window
   - Consider whether the private key's exposure means anything else needs attention (was the key stored anywhere else, accessible to anything else)

5. **Proceed to reissue** if this revocation was to clear the way for a new cert on the same CN, see [Issue a Leaf Certificate](issue-leaf-certificate.md).

---

## Verifying revocation took effect

```bash
openssl ca -config <ca-config-path> -status <serial-number>
```

Confirms the certificate's status in the CA's own records. This is a records-level check, it doesn't confirm any particular client is actually honoring the revocation (see the CRL-checking caveat above).

---

## Common mistakes

- **Revoking the wrong certificate** because the CN or serial wasn't confirmed first, especially easy to get wrong if a service has had multiple certs issued over time.
- **Assuming revocation alone immediately stops a compromised cert from being trusted everywhere**, without confirming whether the relevant clients actually check CRLs (or an equivalent mechanism) rather than just chain-of-trust.
- **Forgetting this step entirely before reissuing**, the mismatched cert/key pair failure mode documented in [Issue a Leaf Certificate](issue-leaf-certificate.md) traces back to skipping this.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Issue a Leaf Certificate](issue-leaf-certificate.md)

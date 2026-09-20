# Runbook: Bring the Root CA Online (Internal PKI)

| Field | Value |
| --- | --- |
| Applies to | Internal PKI (Root CA) |
| Category | Certificate Management |
| Author | Daniel Collins |

---

## When to use this

The Root CA VM is kept **shut down when not in use**, this is a deliberate operational posture, not an oversight, it's only started to perform actual CA-level operations. Use this runbook whenever the Root CA genuinely needs to be online: signing or renewing the Intermediate CA's certificate, or another Root-level operation that can't be done by the Intermediate CA itself. Day-to-day leaf certificate issuance never requires this, that's entirely handled by the online Intermediate CA, see [Issue a Leaf Certificate](issue-leaf-certificate.md).

---

## Before you start

- **Confirm the operation actually requires the Root CA specifically.** The Root CA signs only the Intermediate CA's certificate, nothing else. If the actual need is a leaf cert for some service, this is the wrong runbook, the Root CA should stay off.
- **Have the specific operation planned out before starting the VM.** Minimize how long the Root CA is actually powered on, don't start it and then figure out what to do.

---

## Steps

### 1. Start the Root CA VM

Start it from Proxmox as normal. Confirm it comes up cleanly and is reachable before proceeding.

### 2. Perform the required operation

**Signing/renewing the Intermediate CA certificate:**

```bash
openssl x509 -req -in intermediate-ca.csr \
  -CA root-ca.crt -CAkey root-ca.key \
  -CAcreateserial -out intermediate-ca.crt \
  -days <validity-period> -sha256 \
  -extfile <(printf "basicConstraints=CA:TRUE\nkeyUsage=critical,keyCertSign,cRLSign")
```

Adjust to match this environment's actual Intermediate CA CSR and desired validity period. The `basicConstraints=CA:TRUE` extension is what makes the resulting cert usable as an intermediate signing authority rather than a leaf cert, don't omit it.

**Other Root-level operations** (e.g. generating a Root CRL, if this environment's chain-of-trust model is ever extended to check it) follow the same general pattern: perform only the specific operation needed, nothing else.

### 3. Verify the result before shutting back down

```bash
openssl verify -CAfile root-ca.crt intermediate-ca.crt
```

Confirm the new/renewed Intermediate CA cert actually validates against the Root before considering the operation complete, don't shut down the Root CA and discover a problem afterward when it's already off again.

### 4. Distribute the result

If this was an Intermediate CA renewal, the new Intermediate cert needs to reach every service that references the CA chain (most of them do, for validating leaf certs). This is a broader distribution task than a single service's leaf cert, plan for it rather than treating it as a one-off copy.

### 5. Shut the Root CA back down

Once the operation is verified complete and any necessary distribution is underway, shut the VM back down. Don't leave it running longer than the operation required, this is the entire point of keeping it offline by default.

---

## Common mistakes

- **Starting the Root CA for something the Intermediate CA could have handled**, unnecessarily expands the Root's online exposure window for no reason.
- **Leaving the Root CA running after the operation is done** because something else came up, or out of convenience for "just in case I need it again soon." If another operation is genuinely needed soon, that's still a separate, deliberate start, not an excuse to leave it up.
- **Not verifying the result before shutting back down**, discovering an issue only after the Root CA is off again means starting it up a second time instead of catching it in the same session.
- **Forgetting to redistribute an updated Intermediate CA cert** after a renewal, services can keep validating against the old (soon to expire) Intermediate cert if the new one isn't actually propagated.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Issue a Leaf Certificate](issue-leaf-certificate.md)
- [Distribute the Trust Store](distribute-trust-store.md)

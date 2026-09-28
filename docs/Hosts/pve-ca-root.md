---
name: pve-ca-root
type: vm
vmid: 500
node: pve-services
ip: 10.0.30.21
vlan: VLAN30
status: live
role: Root CA — offline except for CA operations
---

**Login:** `admin`, password auth (SSH key not yet copied as of Sep 25, 2026).

Trust anchor for the entire environment. Signs only the [[pve-ca-intermediate]] certificate, no other purpose for its private key.

**Operational posture:** shut down when not in use, only started for CA operations (signing/renewing the Intermediate CA cert).
**Key algorithm:** RSA 4096 or Ed25519.

Migrated onto VLAN30 (10.0.30.21) from the flat 10.0.0.x network on Sep 20, 2026; previously ~10.0.0.20 (unconfirmed).

The Root CA certificate must be in a client's trust store for that client to trust anything issued by the Intermediate CA.

## Related
- [[pve-services]]
- [[pve-ca-intermediate]]
- [[VLAN30]]

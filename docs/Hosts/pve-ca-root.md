---
name: pve-ca-root
type: vm
vmid: 500
node: pve-services
ip: 10.0.30.x
vlan: VLAN30
status: live
role: Root CA — offline except for CA operations
---

**Login:** `[admin-user]`, password auth (SSH key not yet copied as of Sep 25, 2026).

Trust anchor for the entire environment. Signs only the [[pve-ca-intermediate]] certificate, no other purpose for its private key.

**Operational posture:** shut down when not in use, only started for CA operations (signing/renewing the Intermediate CA cert).
**Key algorithm:** RSA 4096 or Ed25519.

Migrated onto VLAN30 (10.0.30.x) from the flat 10.0.0.x network on Sep 20, 2026; previously ~10.0.0.x (unconfirmed).

The Root CA certificate must be in a client's trust store for that client to trust anything issued by the Intermediate CA.

## DNS (Oct 31, 2026)
Not changed. Daniel confirmed this VM is off at all times unless needed for a specific task. Its resolver setting was never read. When it is next powered on for CA work, check its resolver and point it at 10.0.30.x if it still uses Pi-hole. Nothing in [[dns-phase1-consolidation]] depends on it.

## Related
- [[pve-services]]
- [[pve-ca-intermediate]]
- [[VLAN30]]

## Update Nov 9, 2026: root certificate facts
Subject `C=US, ST=Texas, L=Mansfield, O=Homelab, OU=Certificate Authority, CN=" Homelab Root CA" (leading space), emailAddress=admin@homelab.local`. Valid Dec 27, 2025 to Dec 22, 2045. SHA256 fingerprint `02:01:EF:DC:5D:A7:D7:8B:8A:EA:D2:BF:EF:76:EB:28:16:FE:EE:09:CC:5C:E7:F8:3F:24:AB:D7:69:D7:7E:45`. The leading space in the CN stays, because changing it means replacing the root. See [[session-2026-11-09-stepca-intermediate-rotation]].

## Update Nov 9, 2026: powered on to sign the new step-ca intermediate
Started with `qm start 500` from pve-services, reached by SSH from pve-services at 10.0.30.x (user `[admin-user]`, password). Clock was correct on boot. CA files: `/home/[admin-user]/homelab-ca/root-ca/private/root-ca.key` and `/home/[admin-user]/homelab-ca/root-ca/certs/root-ca.crt`. No `openssl*.cnf` under the home directory to depth 4, so signing is done with `openssl x509 -req`, as in [[pki-bring-root-ca-online]]. Resolver read the same day: `192.168.100.x` (Pi-hole) then `1.1.1.1`, search `homelab.local`. Not changed, to keep the root's online time short. Change it to `10.0.30.x` the next time it is on, and before the Pi-hole DNS rules on VLAN30 are removed, or the root will have no DNS. Details in [[session-2026-11-09-stepca-intermediate-rotation]].

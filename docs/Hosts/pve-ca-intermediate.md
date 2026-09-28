---
name: pve-ca-intermediate
type: vm
vmid: 501
node: pve-services
ip: 10.0.30.22
vlan: VLAN30
status: live
role: Intermediate CA — issues leaf certificates
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Handles all day-to-day certificate issuance for the homelab, online continuously. Signs leaf certificates only (never other CA certs). Runs raw OpenSSL manually (genrsa/req/ca against a hand-rolled CA directory) — no step-ca/EJBCA/ACME on this VM.

Migrated onto VLAN30 (10.0.30.22) from the flat 10.0.0.x network on Sep 20, 2026; previously 10.0.0.21 (confirmed).

Known issue: the current intermediate cert's CN has an errant leading space (`CN=\ Homelab Intermediate CA`, confirmed via `openssl -nameopt RFC2253`). Fix is deferred until cert rollout/revocation is automated.

**Migration in progress:** being replaced by [[pve-int-stepca]] (step-ca, Docker-in-LXC). This VM stays as fallback until the new setup is validated. The real root/intermediate cert and key have already been imported into step-ca's Docker volume (verified via matching SHA256 root fingerprint).

## Issues certificates for
- Elasticsearch/Kibana ([[soc-stack]])
- Wazuh agent enrollment ([[wazuh-manager]])
- Nginx Proxy Manager
- All four Proxmox nodes (mgmt web UI/API)

## Related
- [[pve-services]]
- [[pve-ca-root]]
- [[pve-int-stepca]]
- [[VLAN30]]

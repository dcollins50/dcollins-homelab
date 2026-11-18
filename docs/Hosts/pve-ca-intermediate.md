---
name: pve-ca-intermediate
type: vm
vmid: 501
node: pve-services
ip: 10.0.30.x
vlan: VLAN30
status: live
role: Intermediate CA — issues leaf certificates
---

**Login:** `[admin-user]`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

Handles all day-to-day certificate issuance for the homelab, online continuously. Signs leaf certificates only (never other CA certs). Runs raw OpenSSL manually (genrsa/req/ca against a hand-rolled CA directory) — no step-ca/EJBCA/ACME on this VM.

Migrated onto VLAN30 (10.0.30.x) from the flat 10.0.0.x network on Sep 20, 2026; previously 10.0.0.x (confirmed).

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

## Update Oct 29, 2026: same intermediate key as step-ca confirmed
The public key hash of this VM's `intermediate-ca.crt` matches the one in [[pve-int-stepca]]'s `/home/step/certs/intermediate_ca.crt` (SHA256 of the public key, `89a5901a3d8dae78...`). They are the same key pair, so the intermediate private key exists on two online machines. When this VM is retired, destroy its copy of the key and any backups of it. As of this date every leaf cert deployed before step-ca existed still comes from this VM.

## Update Oct 29, 2026: resolver settings checked
`resolvectl status` on ens18: DNS servers `192.168.100.x` ([[heimdall]], Pi-hole) and `1.1.1.1`, DNS domain `homelab.local`. So this VM asks Pi-hole first, not Unbound on the VLAN30 gateway. From this VM neither `stepca.homelab.local` nor `stepca.homelab.internal` resolved (`getent hosts`). The vault says an Unbound override for `stepca.homelab.local` was added Sep 21, 2026, so that override is either gone or not in the path this VM uses. Resolved the same day: the override is in Unbound and works from pve-int-stepca (`getent hosts stepca.homelab.local` returns 10.0.30.x there, which uses 10.0.30.x). This VM never sees it because it asks Pi-hole. `stepca.homelab.internal` does not resolve anywhere yet. Relevant to [[stepca-acme-leaf-issuance]] and [[dhcp-dns-rollout]].

## Update Oct 29, 2026: which certs this CA actually signed (checked live)
`openssl s_client` from pve-services against port 8006 on each Proxmox node and through internal NPM (SNI `elasticsearch.homelab.local`):
- pve-gateway (10.0.0.x:8006): signed by this CA, `CN=gateway.homelab.local`, expires Feb 8, 2027.
- NPM serving `elasticsearch.homelab.local` (10.0.20.x:443): signed by this CA, expires Mar 30, 2027.
- pve-services, pve-env1, pve-env2 (:8006): NOT from this CA. They present Proxmox's default cluster-CA certs, expiring Sep 2027 (pve-services) and Dec 2027 (env1, env2). The "all four Proxmox nodes" claim in the list above is wrong for those three.
- 10.0.10.x:9200 (Elasticsearch direct): timed out after 5 seconds from pve-services (cause not determined), but answered from wazuh-manager on the same VLAN: signed by this CA, `CN=elasticsearch.homelab.local`, expires Mar 31, 2027. A different cert from the one NPM serves for the same name.
Not checked: Wazuh enrollment cert, other NPM proxy host certs. Uptime Kuma does not monitor cert expiry ([[services-host]]).

## Update Oct 31, 2026: NPM certificate inventory (from the NPM Certificates screen, 14:56)
Internal NPM on [[services-host]] holds 13 certificates, all type Custom Certificate and all In Use, one per name. Who signed each one was not checked (only `elasticsearch.homelab.local` was confirmed as signed by this CA on Oct 29). Expiry (as displayed in the browser): **`npm.homelab.local` Jan 7, 2027 7:24 PM and `portainer.homelab.local` Jan 7, 2027 7:14 PM (earliest, earlier than the Feb 8 figure above)**; `env1`, `env2`, `gateway`, `services` (all `.homelab.local`) Feb 8, 2027; `gitea`, `pihole`, `uptime`, `vaultwarden` (`.homelab.local`) Feb 10, 2027; `elasticsearch.homelab.local` Mar 30, 2027; `authentik.homelab.example` Sep 14, 2027 (public name, issuer not checked); `iris.homelab.local` Sep 19, 2027. No `.internal` certificates exist yet. So the earliest known expiry on services is now Jan 7, 2027.

## Update Oct 31, 2026: resolver moved to Unbound
DNS now points at 10.0.30.x (Unbound on the VLAN30 gateway) only. Before: 192.168.100.x (Pi-hole), then 1.1.1.1, which the VLAN30 firewall never allowed. Search domain is still `homelab.local`. Set in `/etc/netplan/50-cloud-init.yaml`, backup at `~/50-cloud-init.yaml.bak-2026-10-31` on the VM. cloud-init is disabled on this VM (`/etc/cloud/cloud-init.disabled`), so the file is not regenerated. Do not remove that flag: `/etc/cloud/cloud.cfg.d/90-installer-network.cfg` still holds the original installer network config (192.168.100.x/24, gateway 192.168.100.x) and would break the VM's network. Resolves `stepca.homelab.internal` (10.0.30.x) and `.homelab.local` names through Unbound. Part of [[dns-phase1-consolidation]]. Applied per Daniel's report that all checks passed, command outputs not pasted.

## Update Nov 9, 2026: intermediate certificate facts
Read from `/home/[admin-user]/homelab-ca/intermediate-ca/certs/intermediate-ca.crt`: subject `emailAddress=admin@homelab.local, CN=" Homelab Intermediate CA" (leading space), OU=Certificate Authority, O=Homelab, ST=Texas, C=US`; RSA 4096, sha256WithRSA; valid Dec 27, 2025 to Dec 25, 2035; `CA:TRUE, pathlen:0`; key usage Digital Signature, Certificate Sign, CRL Sign. A new intermediate key for [[pve-int-stepca]] is planned, see [[session-2026-11-09-stepca-intermediate-rotation]].

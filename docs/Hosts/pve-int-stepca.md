---
name: pve-int-stepca
type: lxc
vmid: 511
node: pve-services
ip: 10.0.30.20
vlan: VLAN30
status: live
role: step-ca (Docker-in-LXC), planned Intermediate CA replacement
---

**Login:** `admin`, password auth. Normally accessed via `pct enter` from [[pve-services]] rather than direct SSH.

Native Debian 13 LXC running Docker, with step-ca as a container (nesting enabled). Built Sep 19, 2026; Docker confirmed working same day (hello-world ran).

Local user "admin" added (LXC previously had no non-root user).

**Migration approach:** import [[pve-ca-intermediate]]'s existing cert/key rather than generate a fresh root, to preserve trust continuity. Real root cert, intermediate cert, and intermediate key have been imported into the Docker volume, overwriting the throwaway bootstrap PKI — verified via matching SHA256 root fingerprint against [[pve-ca-root]]'s actual cert.

**Authentik OIDC provisioner:** registered and confirmed via `step ca provisioner list` (client ID, config endpoint, enableSSHCA true). Required: enabling step-ca remote admin management, an OPNSense Unbound host override for stepca.homelab.local (no DNS record existed initially), a DEV VLAN firewall rule (pve_int_stepca alias → internal_npm alias, port 443), and importing the homelab root CA cert into the container's trust store.

**Next step:** the OIDC provisioner is registered, this is for SSH bastion cert issuance — the bastion ([[pve-bastion]]) was built Sep 21, 2026, but SSH does not use the certs it issues yet, because sshd trust on the bastion is not configured.

## Related
- [[pve-services]]
- [[pve-ca-intermediate]]
- [[pve-ca-root]]
- [[authentik]]
- [[VLAN30]]

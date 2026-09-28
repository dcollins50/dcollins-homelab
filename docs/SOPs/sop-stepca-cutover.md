---
name: sop-stepca-cutover
type: sop
status: in-progress
---

# SOP: step-ca Cutover (Replacing the OpenSSL Intermediate CA)

**Status:** step-ca is running with the real trust chain imported. The actual cutover — pointing services at step-ca instead of the OpenSSL Intermediate CA — has not happened yet.
**Objective:** Replace [[pve-ca-intermediate]]'s manual OpenSSL issuance process with [[pve-int-stepca]] (step-ca, Docker-in-LXC), without breaking trust for any service currently validating certs against the existing chain.

---

## Why this is an SOP and not a runbook

"Migrate the Intermediate CA" isn't one action on one host — it means step-ca itself, Authentik as its OIDC provisioner, and every single service across the homelab that currently trusts certs from VM 501 ([[soc-stack-vm|soc-stack]], [[wazuh-manager]], internal NPM, all four Proxmox nodes). The cutover step is inherently multi-service, even though each individual piece has its own runbook.

## What's already done

1. **pve-int-stepca (LXC 511) built and running** on [[pve-services]], Docker installed and confirmed working via Docker-in-LXC with nesting enabled.
2. **Real trust chain imported**, not a fresh root — the real root cert, intermediate cert, and intermediate private key were copied into step-ca's Docker volume, overwriting the throwaway bootstrap PKI. Verified via matching SHA256 root fingerprint against [[pve-ca-root]]'s actual cert. step-ca is running clean on the **real** trust chain today, not a placeholder.
3. **Authentik OIDC provisioner registered** on step-ca (`enableSSHCA: true`), confirmed via `step ca provisioner list`. This exists specifically to support the SSH bastion's cert issuance later — see [[sop-ssh-bastion-build]] — not general leaf-cert issuance yet.
4. **A known issue carried over, not yet fixed:** the existing intermediate cert's CN has an errant leading space (`CN=\ Homelab Intermediate CA`, confirmed via `openssl -nameopt RFC2253`). The decision was to hold off reissuing the real intermediate cert to fix this until cert rollout and revocation are automated — fixing it manually now would just mean doing the same reissue-and-redistribute work twice.

## What "cutover" actually requires (not yet started)

1. **Decide the leaf-cert issuance workflow on step-ca** — this environment's current process is manual `openssl` commands run locally on the Intermediate CA VM (see [[PKI]]). step-ca supports automated issuance (ACME, or provisioner-based). Which model replaces the manual process hasn't been decided.
2. **Reissue every currently-deployed leaf cert** through step-ca instead of VM 501, for: Elasticsearch/Kibana ([[soc-stack-vm|soc-stack]]), Wazuh agent enrollment ([[wazuh-manager]]), internal NPM, and all four Proxmox node management interfaces.
3. **Validate each service still trusts the chain** after cutover — since step-ca was seeded with the *same* root/intermediate cert VM 501 already uses, existing deployed certs and existing trust stores shouldn't need to change, only newly-issued certs come from step-ca going forward. This needs to be confirmed empirically per service, not assumed.
4. **Keep VM 501 as fallback** until the new setup is validated — this was the original decision and still holds. No target date has been set for retiring it.

## Related Documentation

- [[PKI]]
- [[pve-ca-intermediate]]
- [[pve-ca-root]]
- [[pve-int-stepca]]
- [[authentik]]
- [[pki-stepca-import-existing-ca|Import an Existing CA into step-ca]]
- [[pki-check-cert-subject-formatting|Check a Certificate's Subject Fields for Formatting Issues]]
- [[pki-issue-leaf-certificate|Issue a Leaf Certificate]]
- [[proxmox-docker-reset-container-init|Reset a Docker Container's Init]]

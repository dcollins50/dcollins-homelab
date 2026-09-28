---
name: sop-pki-migration-vlan30
type: sop
status: complete
---

# SOP: PKI Migration to Trust Infrastructure VLAN (VLAN30)

**Status:** Complete (Sep 20, 2026)
**Objective:** Move the internal Root and Intermediate CA off the flat 10.0.0.x network onto a dedicated, scoped VLAN, without breaking any service that depends on them for TLS.

---

## Why this was multi-app, not a single runbook

This touched OPNSense (new aliases, new scoped firewall rules), the two CA VMs themselves (IP reassignment), and every downstream service that validates certs against this chain. No single tool's runbook covers "move a trust anchor to a new network segment and confirm nothing broke."

## Background

[[pve-ca-root]] and [[pve-ca-intermediate]] had been running on the flat 10.0.0.x scheme, riding a full trunk port with no VLAN tag set on `net0`, despite documentation already assuming they were on [[VLAN30]]. This was discovered Sep 14, 2026 during the Authentik/DEV VLAN rework, when VLAN30 was confirmed as the switch's actual "Dev" VLAN ID (not 6, as OPNSense's `vlan06` interface naming had suggested).

## Steps taken

1. **Confirmed the correct VLAN ID.** Switch VLAN table: 1 Default, 10 SOC, 20 Services, 30 Dev (Trust Infrastructure), 40/41 Security Lab, 50 DMZ, 60 Storage.
2. **Created host aliases ahead of the migration**, rather than relying on the existing blanket "Allow LAN to DEV" rule: `pve_ca_root` and `pve_ca_intermediate`.
3. **Added scoped workstation-to-host SSH rules** for each alias, narrower than the blanket rule.
4. **Reassigned IPs:**
   - [[pve-ca-root]]: 10.0.0.20 (unconfirmed) → **10.0.30.21**
   - [[pve-ca-intermediate]]: 10.0.0.21 (confirmed) → **10.0.30.22**
5. **Verified the migration:**
   - SSH reachable at both new addresses
   - Old addresses unreachable
   - Confirmed no hardcoded references to the old IPs anywhere else — cert issuance has always been manual `openssl` commands run locally on each CA VM, so there was no config drift risk from the IP change itself.

## Result

Both CAs now sit on [[VLAN30]] alongside [[authentik]] and [[pve-int-stepca]], as originally intended. See [[PKI]] for the CA hierarchy and [[VLAN30]] for the current VLAN's host table and firewall policy.

## Related Documentation

- [[PKI]]
- [[VLAN30]]
- [[pve-ca-root]]
- [[pve-ca-intermediate]]
- [[opnsense-add-firewall-rule|Add a Firewall Rule]]
- [[opnsense-create-manage-aliases|Create and Manage Aliases]]

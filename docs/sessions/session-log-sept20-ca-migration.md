# Session Log — September 20, 2026 (Evening): PKI Migration to Trust Infrastructure VLAN

**Scope:** Migrating the Root CA and Intermediate CA off the flat 10.0.0.x network and onto VLAN30 (Trust Infrastructure), closing the open item flagged in [Session Log — September 20, 2026 (PM)](session-log-sept20-documentation-rebuild.md) and originally identified back in [Session Log — September 14, 2026](session-log-sept14-authentik-devvlan.md).

---

## Pre-migration: Aliases and Firewall Rules

Before touching either VM, two new aliases and their matching access rules were put in place so connectivity wouldn't break mid-migration:

**Aliases** (Firewall → Aliases → Aliases):

| Name | Type | Content | Description |
|---|---|---|---|
| `pve_ca_root` | Host(s) | 10.0.30.21 | Root CA host |
| `pve_ca_intermediate` | Host(s) | 10.0.30.22 | Intermediate CA host |

**Firewall rules** (LAN interface tab, matching the existing `pve-iris`/`soar-host` pattern rather than relying on the pre-existing blanket `Allow LAN to DEV` rule):

| Rule | Source | Destination | Port |
|---|---|---|---|
| Allow workstation admin of pve-ca-root | Workstation | `pve_ca_root` | 22 (SSH) |
| Allow workstation admin of pve-ca-intermediate | Workstation | `pve_ca_intermediate` | 22 (SSH) |

Both added and applied ahead of the actual migration, so SSH access to the new addresses would be live the moment each VM came back up on VLAN30.

---

## Migration Procedure

Performed on both VMs, Root CA first:

1. Powered the VM on (on its existing flat-network address).
2. Edited the guest's netplan config, updating the static IP to its assigned VLAN30 address and the corresponding VLAN30 gateway.
3. Saved the netplan change, then shut the VM down cleanly.
4. In Proxmox, changed the VM's network device (`net0`) VLAN tag to 30.
5. Powered the VM back on.

Repeated for both:

| Host | Old address | New address |
|---|---|---|
| pve-ca-root | 10.0.0.20 (assumed, never explicitly confirmed pre-migration) | 10.0.30.21 |
| pve-ca-intermediate | 10.0.0.21 (confirmed) | 10.0.30.22 |

Both came up correctly on the new VLAN and address on the first boot after the tag change, no rollback or troubleshooting needed.

---

## Verification

- **SSH connectivity confirmed** to both new addresses (10.0.30.21, 10.0.30.22).
- **Old addresses confirmed unreachable** (10.0.0.20, 10.0.0.21 both stopped responding post-migration).
- **No Wazuh agent reconnection to check**, neither VM runs its own Wazuh agent, monitoring visibility for both comes from pve-services' host-level agent, which never moved and was unaffected.
- **No hardcoded IP references elsewhere in the environment.** Certificate issuance on both CAs has always been manual `openssl` commands run locally on the CA host itself, nothing external (scripts, cron jobs, other configs) referenced either CA's old flat-network address, so there was nothing left dangling to find and fix.

---

## Result

Both CAs are now actually on VLAN30 (Trust Infrastructure), not just documented as being there. [Internal PKI](../../pki.md) and [Infrastructure](../../infrastructure.md) were written ahead of this migration (with real octets redacted as `10.0.30.x`), that documentation is now accurate rather than aspirational.

## Open items carried forward

- Root CA's actual pre-migration address (10.0.0.20) was never independently confirmed, it was inferred from Intermediate CA's known address (10.0.0.21) and is now moot since the migration is complete, but worth noting for the record that it was an assumption, not a verified fact, at the time.
- The blanket `Allow LAN to DEV` rule on VLAN30 is still broader than the scoped, per-host pattern used elsewhere (VLAN10's pve-iris/soar-host rules). The two new CA-specific SSH rules added this session are redundant with that blanket rule for now. Narrowing or removing the blanket rule remains a separate, not-yet-scheduled hardening task.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Infrastructure](../../infrastructure.md)
- [Session Log — September 14, 2026: Authentik / DEV VLAN Rework](session-log-sept14-authentik-devvlan.md)
- [Session Log — September 20, 2026 (PM): Documentation Rebuild](session-log-sept20-documentation-rebuild.md)
- [Add a Firewall Rule](../runbooks/opnsense/add-firewall-rule.md)

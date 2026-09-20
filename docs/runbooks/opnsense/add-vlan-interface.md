# Runbook: Add a VLAN Interface (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense, managed switch, Proxmox |
| Category | Network Segmentation |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new isolated network segment is needed, a new class of workload that shouldn't share broadcast domain or firewall policy with anything existing. This is the end-to-end task version; for the full first-time buildout reasoning, failure modes, and recovery procedures, see [sop-vlan-implementation.md](../../sop/sop-vlan-implementation.md). This runbook assumes the switch and OPNSense are already VLAN-capable and configured, and just walks the repeatable steps for adding one more VLAN to that existing setup.

---

## Before you start

- **Pick the VLAN ID.** Check [network.md](../../network.md) for what's already in use before picking a new one.
- **Pick the subnet and gateway IP.** Must not overlap any existing VLAN or the management network.
- **Know the purpose.** What will live here, and what does it need to reach (and be reached by)? This determines the firewall policy in the last step, so have it decided before touching the firewall, not worked out ad hoc while writing rules.
- **Confirm switch port plan.** Which physical ports need this VLAN tagged, and are any of them shared trunk ports carrying multiple VLANs already.

---

## Steps

### 1. Create the VLAN interface in OPNSense

Interfaces → Other Types → VLAN → `+` Add.

- Parent interface: the LAN-side physical interface (or the interface that trunks to the switch)
- VLAN tag: the ID chosen above
- Description: a short name for the VLAN's purpose

Save.

### 2. Assign the new VLAN device as an interface

Interfaces → Assignments.

- Select the new VLAN device from the dropdown, click `+` to assign it (creates a new OPTx interface)
- Save

### 3. Configure the new interface

Interfaces → [OPTx].

- Enable the interface
- Description: match the VLAN's purpose
- IPv4 Configuration Type: Static
- IPv4 Address: the gateway IP chosen above, with the appropriate subnet mask
- Leave Gateway blank (this is an internal interface, not the default route)

Save, then Apply Changes.

### 4. Configure VLAN membership on the managed switch

Log into the switch's management interface.

- Create the VLAN (matching ID) if the switch requires explicit VLAN creation separate from port tagging
- For each port that needs to carry this VLAN:
  - **Tagged**, if the port is a trunk carrying multiple VLANs (e.g. to OPNSense, to a Proxmox node)
  - **Untagged**, if the port is a dedicated access port for a single device that isn't VLAN-aware itself
- Apply the switch configuration

### 5. Enable VLAN awareness on the Proxmox side (if VMs will live on this VLAN)

On the relevant Proxmox node(s), confirm the bridge is VLAN-aware (`bridge-vlan-aware yes` on the bridge used by VMs on this segment). This is typically already enabled cluster-wide once done for the first VLAN, so this step is usually just a verification, not a new change.

For each VM or container that needs to live on the new VLAN, set its network device's VLAN tag to match.

### 6. Add baseline firewall rules

Firewall → Rules → [new interface tab].

Start from default-deny and add only what the VLAN's actual purpose requires: see [Add a Firewall Rule](add-firewall-rule.md). At minimum, most VLANs need:

- DNS resolution path (permitted to the environment's resolver, blocked elsewhere)
- Whatever specific inter-VLAN or internet access the VLAN's purpose actually calls for, and nothing broader

Do not add a blanket "allow to any" rule unless the VLAN's purpose genuinely requires unrestricted outbound access.

### 7. Test connectivity

- From a host or VM placed on the new VLAN, confirm it gets the correct IP and can reach its gateway
- Confirm DNS resolution works as expected
- Confirm the specific access paths added in step 6 work, and that nothing broader than intended is also passing
- Confirm the VLAN cannot reach anything it shouldn't (test a path that should be blocked, don't just test the ones that should work)

### 8. Document it

Update [network.md](../../network.md) with the new VLAN's entry: purpose, hosts, firewall policy. Update the switch's VLAN list if this doc tracks it. If this was a substantial segmentation change rather than a small addition, consider whether it's worth a session log.

---

## Common mistakes

- **Trunk vs. access port confusion on the switch** is the single most common cause of a new VLAN silently not working, double check tagged vs. untagged per port rather than assuming.
- **Forgetting the VLAN-aware bridge check** on Proxmox when VMs are involved, traffic looks correctly configured everywhere else but never leaves the host.
- **Skipping the "confirm it's blocked where it should be" test.** A VLAN that can reach something unintended is a bigger problem than one that can't reach something it needs, and it's easy to only test the paths you expect to work.
- **Adding rules broader than the VLAN's actual purpose** just to get something working quickly, then never narrowing them back down.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [SOP: VLAN Implementation](../../sop/sop-vlan-implementation.md)
- [Add a Firewall Rule](add-firewall-rule.md)

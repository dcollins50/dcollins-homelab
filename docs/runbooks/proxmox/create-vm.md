# Runbook: Create a VM (Proxmox)

| Field | Value |
| --- | --- |
| Applies to | Proxmox VE |
| Category | Provisioning |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new workload needs a full VM rather than a container, generally anything that needs its own kernel, a different OS than the host, or isolation stronger than an LXC provides.

---

## Before you start

- **Check node resource availability before deploying**, not after. Confirm the target node has enough free RAM and CPU headroom for the new VM's planned allocation, don't assume based on the node's total spec, check actual current usage.
- **Decide the VM's network placement**: which VLAN, and therefore which bridge and VLAN tag.
- **Decide the VMID.** Keep it consistent with whatever numbering convention is already in use for the category of workload (see [Infrastructure](../../infrastructure.md) for the current VM inventory and ID ranges in use).
- **Have the install media or template ready** (ISO uploaded, or a template you're cloning from, if cloning, use [Create an LXC](create-lxc.md)'s sibling logic doesn't apply here, VM cloning is a separate flow from this fresh-install one).

---

## Steps

1. **Select the target node** in the Proxmox UI, then **Create VM** (top right).
2. **General tab:**
   - VMID: per the convention above
   - Name: descriptive, matching the naming pattern already in use
3. **OS tab:** select the ISO/install media, and the correct OS type/version for accurate default settings.
4. **System tab:** defaults are usually fine; adjust BIOS/machine type only if the workload specifically requires it (e.g. UEFI for certain guests).
5. **Disks tab:**
   - Select the correct storage target
   - Size the disk appropriately for the workload, don't just default to a round number without thinking about what it'll actually hold
6. **CPU tab:** set core count based on actual need and current node headroom, not the max available.
7. **Memory tab:** same principle, allocate based on the workload's real requirement plus reasonable margin, not the max the node has free.
8. **Network tab:**
   - Bridge: the bridge serving the target VLAN
   - VLAN Tag: the VLAN ID for this workload's placement (see [Set a VM's VLAN Tag](set-vlan-tag-vm-network.md) if this is being changed after creation instead of set here)
   - Firewall checkbox: match the convention already in use in this environment for VM-level firewall (this environment has generally left VM-level firewall disabled in favor of OPNSense enforcing policy at the VLAN boundary, confirm this is still the intended approach before changing it)
9. **Confirm tab:** review all settings before finishing, this is the last easy point to catch a mistake before the VM exists.
10. **Start the VM** and complete the OS installation.
11. **Post-install:**
    - Set a static IP matching the VLAN's addressing plan (or confirm DHCP reservation if that's the convention for this VLAN)
    - Confirm the VM can resolve DNS and reach its expected gateway
    - Deploy the Wazuh agent if this VM is meant to be monitored (standard practice for workloads outside the isolated lab VLANs)
12. **Document it.** Add the new VM to [Infrastructure](../../infrastructure.md)'s VM inventory table for its host node, and to any other doc where that node's/VLAN's inventory is tracked (e.g. [SOC Stack](../../soc-stack.md) if it's a SOC-related VM).

---

## Common mistakes

- **Not checking actual node headroom first**, leading to over-committing a node's resources across several VMs that each look fine individually.
- **Forgetting the VLAN tag on the network device**, the VM comes up on the default/untagged network instead of its intended segment, silently exposing it to the wrong broadcast domain.
- **Skipping the documentation step**, leading to the same VM-inventory drift this repo has already had to clean up more than once.

---

## Related Documentation

- [Infrastructure](../../infrastructure.md)
- [Create an LXC](create-lxc.md)
- [Add a Firewall Rule](../opnsense/add-firewall-rule.md)

# Runbook: Create an LXC (Proxmox)

| Field | Value |
| --- | --- |
| Applies to | Proxmox VE |
| Category | Provisioning |
| Author | Daniel Collins |

---

## When to use this

Use this for a single-application internal service where a full VM's overhead isn't warranted. This environment's stated preference is LXCs over Docker hosts for single-application internal services where practical (Authentik, step-ca, the internal CAs all run this way), reach for this before spinning up a general-purpose Docker VM for something that only needs to run one thing.

---

## Before you start

- **Check node resource availability before deploying**, same as any other provisioning task, don't assume headroom, check it.
- **Decide the VM/LXC ID** (LXCs and VMs share the same ID namespace in Proxmox) per the existing numbering convention, see [Infrastructure](../../infrastructure.md) for current usage.
- **Decide the VLAN placement.**
- **Decide the base template.** This environment's LXCs have consistently used Debian templates, stick with that unless there's a specific reason not to, for consistency across the fleet.
- **Decide whether Docker-in-LXC is needed.** If the workload is more naturally run as a Docker container (e.g. an image-based deployment) rather than a native package install, this environment's precedent (step-ca) is Docker-in-LXC with nesting enabled, rather than a dedicated Docker VM, when the workload is still fundamentally single-purpose.

---

## Steps

1. **Select the target node**, then **Create CT** (Container).
2. **General tab:**
   - CT ID: per the convention above
   - Hostname: descriptive, matching naming pattern already in use
   - Set a password or SSH key for root access
3. **Template tab:** select the Debian template (or the specific base image decided above).
4. **Disks tab:** size appropriately for a single-application workload, LXCs typically need far less than a comparable VM.
5. **CPU tab:** allocate based on actual need.
6. **Memory tab:** same, allocate based on actual need plus reasonable margin.
7. **Network tab:**
   - Bridge: the bridge serving the target VLAN
   - VLAN Tag: the VLAN ID for this workload
   - IPv4: static, matching the VLAN's addressing plan, or DHCP if that's the convention in use
8. **DNS tab:** confirm it's set to resolve correctly for the target VLAN (usually inherited correctly, but worth confirming rather than assuming).
9. **Confirm tab**, review, then finish.
10. **Start the container.**
11. **If this LXC will run Docker:**
    - Install Docker inside the container
    - Ensure **nesting is enabled** on the container (Options → Features → Nesting), Docker-in-LXC will not work correctly without it
    - Confirm Docker actually runs (e.g. `docker run hello-world`) before building anything further on top of it
12. **Post-install:**
    - Confirm the container can resolve DNS and reach its expected gateway
    - Deploy the Wazuh agent if applicable to this workload's monitoring needs
13. **Document it.** Add the new LXC to [Infrastructure](../../infrastructure.md)'s inventory table for its host node, and anywhere else that node's/VLAN's inventory is tracked.

---

## Common mistakes

- **Defaulting to a Docker VM out of habit** when the workload is genuinely single-purpose and would fit this environment's LXC-first convention better.
- **Forgetting to enable nesting** before trying to run Docker inside the container, leads to a confusing failure that looks like a Docker problem but is actually a container feature flag.
- **Under-sizing disk for a Docker-in-LXC setup**, image layers and volumes add up faster than a native package install would.

---

## Related Documentation

- [Infrastructure](../../infrastructure.md)
- [Internal PKI](../../pki.md)
- [Create a VM](create-vm.md)

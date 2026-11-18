---
name: pve-services
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.x
vlan: VLAN1 (mgmt), VLAN30 (trust infra), VLAN50 (DMZ), VLAN51 (bastion), VLAN52 (DMZTunnels)
status: live
role: Internal services, PKI and trust infrastructure
---

HP EliteDesk G3. One of four Proxmox nodes, Corosync quorum member. 4 cores (i5-7500), 15.50 GiB RAM (5.76 GiB used on Nov 7, 2026).

**Version:** Proxmox VE 9.2.26 / kernel 7.0.14-28-pve since Nov 8, 2026 (was 9.1.1 / 6.17.2-1-pve). Kernels 6.17.2-1-pve and 6.17.13-21-pve are still installed as fallbacks; none pinned. Confirmed Nov 9, 2026 with `cat /proc/cmdline` that the NVMe workaround parameters below carried over to kernel 7.0.

**Package repositories:** Proxmox No-Subscription on `trixie`, added Nov 8, 2026. Before that the only Proxmox repository was the Proxmox VE 8 (`bookworm`) entry in `pve-no-subscription.list`, so the node got no Proxmox updates and showed "No Proxmox VE repository enabled". That entry is disabled, not deleted. `wazuh-agent` and `filebeat` are on `apt-mark hold` until the SOC update. See [[homelab-update-automation]].

**Backups:** none scheduled. The only copies of its guests are the one-off vzdump backups from Oct 17, 2026.

**System disk:** Samsung SSD 990 PRO 1TB NVMe, firmware 5B2QJXD7, installed around mid-April 2026. Boots through plain GRUB (`proxmox-boot-tool` not in use). Since Oct 17, 2026 the kernel runs with `nvme_core.default_ps_max_latency_us=0 pcie_aspm=off` to disable drive power saving, after [[incident-2026-10-15-pve-services-nvme-dropout]].

**Time:** time zone UTC ([[standardize-node-clocks-utc]]). Synced by chrony against public NTP servers, not OPNsense. Checked Oct 17, 2026 with `chronyc sources -v`: four sources, all reachable (Reach 377): `LAX1.CALTICK.NET` (current best), `ntp.maxhost.io`, `hou1.us.ntp.li`, `[public-ip]`; offsets within about 10 ms. Its LXCs (including [[pve-bastion]] and [[pve-stepca-tunnel]]) share this clock: they show `NTP service: inactive` and `System clock synchronized: yes`, so they need no NTP firewall rule.

## Hosted VMs/LXCs
- [[ubuntu-401]] (VM 401)
- [[pve-ca-root]] (VM 500)
- [[pve-ca-intermediate]] (VM 501)
- [[authentik]] (LXC 2201)
- [[pve-int-stepca]] (LXC 511)
- [[pve-bastion]] (LXC 2202)
- [[pve-authtunnel]] (LXC 2100)
- [[pve-stepca-tunnel]] (LXC 2101)

## Related
- [[VLAN1]]
- [[VLAN30]]
- [[VLAN51]]
- [[VLAN52]]

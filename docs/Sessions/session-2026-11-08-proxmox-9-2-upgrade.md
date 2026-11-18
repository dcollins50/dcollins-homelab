---
name: session-2026-11-08-proxmox-9-2-upgrade
type: session
date: 2026-11-08
---

Started as groundwork for [[homelab-update-automation]] and turned into the cluster's first real update round.

## Found
- The pve-services Summary page showed "No Proxmox VE repository enabled!". Checking all four nodes' Repositories pages:
  - [[pve-gateway]] and [[pve-services]]: only Proxmox repository was the Proxmox VE 8 `bookworm` No-Subscription entry, left over from the 8 to 9 upgrade.
  - [[pve-env1]] and [[pve-env2]]: paid `pve-enterprise` and Ceph enterprise repositories on, no subscription.
  - So no node had been getting Proxmox updates since moving to 9.x.
- After fixing the repositories, every node had the Proxmox VE 9.1 to 9.2 upgrade pending (pve-manager 9.1.1 to 9.2.26, kernel 7.0, QEMU 11, LXC 7.0, ZFS 2.4, corosync 3.2), plus Debian security updates. pve-gateway, pve-env1 and pve-env2 were further behind on Debian packages (openssl, curl, bind9, libexpat1, rsync) than pve-services.
- wazuh-agent was 4.14.7 to 4.14.9 across nodes, filebeat 8.19.21 to 8.19.23.

## Decided
- Order: pve-gateway, pve-env1, pve-env2, pve-services.
- SOC pipeline updated separately: `wazuh-agent` and `filebeat` held on every node.
- [[malware-win11]] is rebuildable, no backup before the reboot.

## Done
- Repositories fixed on all four ([[proxmox-fix-package-repositories]]).
- `apt full-upgrade` run on all four with the holds in place (all at once, not one node at a time as planned). Answered No to the shim Secure Boot prompt.
- Checked all four before rebooting: quorate 4 of 4, 9.2.26 installed, kernels 6.17.2-1, 6.17.13-21 and 7.0.14-28 present, no failed units, holds confirmed.
- Rebooted pve-gateway, then pve-env1 (both checked: 7.0.14-28-pve, 9.2.26, quorate, no failed units), then pve-env2 and pve-services. Daniel reports everything back online.
- Procedure written up as [[proxmox-upgrade-node]].
- Nov 9, 2026: Daniel confirmed all logs flowing into Kibana after the pve-env2 reboot restarted soc-stack and wazuh-manager.
- Nov 9, 2026: malware-win11 boots and accepts login on QEMU 11 / kernel 7.0. All guests that need Start at boot have it enabled.
- Nov 9, 2026: pve-services boots `7.0.14-28-pve` with `nvme_core.default_ps_max_latency_us=0 pcie_aspm=off` still on the command line.

## Not confirmed
- pve-services rebooted without the fresh guest backups that were proposed. Scheduled backups for pve-services tabled Nov 9, 2026 as a separate project.

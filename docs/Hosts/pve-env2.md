---
name: pve-env2
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.x
vlan: VLAN1 (mgmt), VLAN10 (SOC)
status: live
role: SOC stack — dedicated to ELK and Wazuh
---

HP EliteDesk G6. One of four Proxmox nodes, Corosync quorum member. 32GB RAM total.

**Storage** (Proxmox web UI, Disks, Nov 16, 2026):
- `nvme0n1`: Samsung SSD 990 PRO 1TB, serial S7LANU0Y707540T, SMART passed, 2% wearout. System disk (BIOS boot, EFI, LVM). Same model as the [[pve-services]] drive in [[incident-2026-10-15-pve-services-nvme-dropout]]; whether this node has the same power-saving workaround is not checked.
- `nvme1n1`: WDC PC SN720 256 GB (238.5 GiB), serial 202118807055, SMART passed, 7% wearout. One ext4 partition, passed through to the SOC stack and storing the Wazuh/ELK log data (search-engine data directory, about 32 GB on Nov 16, 2026). Not a Proxmox storage entry and not mounted on the host. **In use, not spare.** Previously misfiled in the vault under [[pve-env1]] as available.

**Version:** Proxmox VE 9.2.26 / kernel 7.0.14-28-pve since Nov 8, 2026 (was 9.1.1 / 6.17.2-1-pve). Kernels 6.17.2-1-pve and 6.17.13-21-pve are still installed as fallbacks; none pinned.

**Package repositories:** Proxmox No-Subscription on `trixie`, added Nov 8, 2026. Before that the paid `pve-enterprise` and Ceph enterprise (`ceph-squid`) repositories were on with no subscription, so Proxmox updates failed. Both are disabled, not deleted. `wazuh-agent` and `filebeat` are on `apt-mark hold` until the SOC update. See [[homelab-update-automation]].

Rebooted Nov 8, 2026 for the kernel, which restarted [[soc-stack-vm|soc-stack]] and [[wazuh-manager]]. Daniel confirmed all logs flowing into Kibana again on Nov 9, 2026.

Reserved exclusively for the core SOC stack (Elastic, Wazuh). Remaining capacity is reserved for additional agents, not new services — that's why soar-host and pve-iris run on [[pve-env1]] instead.

## Hosted VMs
- [[soc-stack-vm|soc-stack]] (VM 600, 10.0.10.x) — Elasticsearch, Logstash, Kibana
- [[wazuh-manager]] (VM 601, 10.0.10.x) — Wazuh SIEM Manager

## Related
- [[VLAN1]]
- [[VLAN10]]

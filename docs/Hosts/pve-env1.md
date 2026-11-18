---
name: pve-env1
type: physical-node
vmid: n/a
node: n/a
ip: 10.0.0.x
vlan: VLAN1 (mgmt), VLAN10 (SOC), VLAN20 (services), VLAN50 (DMZ), VLAN55 (CI)
status: live
role: Production services, Docker workloads, SOC automation
---

HP EliteDesk G5. One of four Proxmox nodes, Corosync quorum member.

**Version:** Proxmox VE 9.2.26 / kernel 7.0.14-28-pve since Nov 8, 2026 (was 9.1.1 / 6.17.2-1-pve). Kernels 6.17.2-1-pve and 6.17.13-21-pve are still installed as fallbacks; none pinned.

**Package repositories:** Proxmox No-Subscription on `trixie`, added Nov 8, 2026. Before that the paid `pve-enterprise` and Ceph enterprise (`ceph-squid`) repositories were on with no subscription, so Proxmox updates failed. Both are disabled, not deleted. `wazuh-agent` and `filebeat` are on `apt-mark hold` until the SOC update. See [[homelab-update-automation]].

**Specs:** 6-core Intel i5-9500 @ 3.00GHz, 31.13 GiB RAM
**Storage:** 931.5GB NVMe (Phison PS5027-E27T controller; primary, Proxmox data pool/VM disks). No second disk: `lspci` shows only one NVMe controller (checked Nov 16, 2026). The 238.5GB secondary NVMe this note used to list is actually in [[pve-env2]].

## Hosted VMs
- [[services-host]] (VM 200, 10.0.20.x) — Docker Host 1
- [[soar-host]] (VM 602, 10.0.10.x) — Shuffle SOAR
- [[docker-host-template]] (VM 603) — clone template
- [[npm-dmz]] (LXC 2200, 10.0.50.x) Cloudflare Tunnel and public reverse proxy
- [[pve-iris]] (VM 604, 10.0.10.x) — DFIR-IRIS
- [[dev-git]] (LXC 3300, 10.0.50.x) Separate Gitea shared to one outside user over Tailscale
- [[nimbus]] (LXC 3400, 10.0.50.x) Nextcloud file storage shared with the collaborator over Tailscale
- [[pve-ansible]] (LXC 1100, 10.0.20.x) Ansible control node for the Achilles SOC build
- [[pve-ci]] (VM 800, 10.0.55.x, [[VLAN55]]) Gitea Actions CI runner for dev-git
- [[pve-pbs]] (VM 6000, 10.0.60.x, [[VLAN60]]) Proxmox Backup Server

## Backup jobs
- One weekly Proxmox backup job for [[dev-git]] (CT 3300) and [[nimbus]] (CT 3400) to `local`, `sun 03:00` (UTC), snapshot mode, ZSTD, keep last 3.
- `/usr/local/bin/devgit-backup-send`: forced command for the workstation's `devgit_backup_pull` key in `admin`'s `authorized_keys`. Lists or sends only `vzdump-lxc-3300-*` and `vzdump-lxc-3400-*` backups (widened Sep 30, 2026; previous copy `devgit-backup-send.bak`). See [[dev-git-backup-strategy]] and [[nimbus-backup-design]].
- `/usr/local/bin/nimbus-data-snapshot`, run by `/etc/cron.d/nimbus-data-snapshot` (`30 3 * * 0`, root): weekly read-only LVM snapshot of nimbus's data disk (`pve/vm-3400-disk-2`), mounted at `/mnt/nimbus-data-snap`. Log `/var/log/nimbus-data-snapshot.log`. The mount does not survive a reboot of this node until the next Sunday run.
- `admin`'s `authorized_keys` also holds the `nimbus-data-pull` key, forced to `sudo /usr/bin/rrsync -ro /mnt/nimbus-data-snap/data`, with `/etc/sudoers.d/nimbus-pull` allowing only that command. Details in [[nimbus-backup-design]].

## Monitoring
- **Thin pool alert (Nov 16, 2026):** `/usr/local/bin/thinpool-check`, run hourly at :17 by `/etc/cron.d/thinpool-check`, reads `lvs` data and metadata percent for `pve/data` and pushes to an Uptime Kuma push monitor `pve-env1 disk pool` (push URL in `/etc/thinpool-check.conf`, mode 600). Status down at 80% or more; Kuma heartbeat interval 4200 s, so a missing push also alerts. Alerts go to ntfy. First push: data 13.95%, meta 0.67%. Reason: guest disks claim 1.22 TiB against the 930 GiB thin pool and thin-pool autoextend is off.
- **Homelab root CA trusted (Nov 16, 2026):** installed at `/usr/local/share/ca-certificates/homelab-root-ca.crt` (SHA-256 fingerprint starts `02:01:EF:DC`), taken from `intermediate-chain.crt` on the workstation. Needed for `https://uptime.homelab.internal`.

## Related
- [[VLAN1]]
- [[VLAN20]]
- [[VLAN50]]
- [[VLAN55]]

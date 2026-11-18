---
name: pve-pbs
type: vm
vmid: 6000
node: pve-env1
ip: 10.0.60.x
vlan: VLAN60 (Storage)
status: live
role: Proxmox Backup Server, local datastore synced to Backblaze B2
---

Proxmox Backup Server 4.2-1 VM on [[pve-env1]], the first host on [[VLAN60]]. Built Nov 16, 2026. Design and scope: [[offsite-backups-pbs-b2]].

## VM (created by script on pve-env1, Nov 16, 2026)
- 4 cores (`x86-64-v2-AES`), 6144 MB RAM, q35, OVMF with pre-enrolled keys, start at boot, QEMU agent enabled
- net0: `vmbr0`, tag 60, no Proxmox firewall
- Disks, all `local-lvm`, discard/iothread/SSD on:
  - scsi0 32 GB: operating system
  - scsi1 250 GB: local datastore
  - scsi2 64 GB: Backblaze B2 datastore cache
- Installer ISO `proxmox-backup-server_4.2-1.iso` (SHA-256 verified) on `local`; detached from the VM after install

## Network
- 10.0.60.x/24, gateway and DNS 10.0.60.x ([[opnsense]])
- Hostname `pve-pbs.homelab.internal` (per [[dns-architecture]])
- Web UI and API: `https://10.0.60.x:8007` (workstation only)
- Firewall rules: see [[VLAN60]]

## Setup done Nov 16, 2026
- Installed Proxmox Backup Server 4.2.0, boots EFI with Secure Boot, kernel 7.0.0-3-pve. Login `root@pam` (password in Vaultwarden, `homelab/hosts/pve-pbs`).
- Repositories: `pbs-enterprise.sources` disabled (`Enabled: false`), No-Subscription added in `/etc/apt/sources.list.d/proxmox.sources` (trixie, `pbs-no-subscription`). `apt full-upgrade` ran clean through the VLAN60 apt rules.
- Disks inside the VM: `sda` 32 GB OS (LVM `pbs-root`, `pbs-swap`), `sdb` 250 GB ext4 at `/mnt/datastore/local-store` = datastore **local-store**, `sdc` 64 GB ext4 at `/mnt/datastore/s3-cache` (B2 cache, not a datastore).

- **Secure Boot turned off** in the VM's firmware (Device Manager > Secure Boot Configuration > Attempt Secure Boot cleared). After the first `apt full-upgrade`, the reboot failed with `Verification failed: (0x1A) Security Violation`: the update added a `debian` boot entry ahead of `proxmox`, and the firmware rejected the new boot chain. Known Proxmox 9.2-era issue reported on the Proxmox forum for fresh Proxmox Backup Server installs updated right after install. Boots normally with Secure Boot off.

- **B2:** S3 endpoint `backblaze-b2` (endpoint `{{bucket}}.s3.{{region}}.backblazeb2.com`, region `us-east-005`, access key = B2 keyID `[b2-key-id]`, provider quirk `skip-if-none-match-header`, required for Backblaze). Datastore **b2-store** on bucket `[b2-bucket]`, local cache `/mnt/datastore/s3-cache/b2-store`. Created Nov 16, 2026, TASK OK. OPNsense rule: Storage interface, `pve_pbs` to alias `b2_endpoint` (the endpoint and the bucket-specific hostname) on TCP 443.

## Jobs (configured Nov 16, 2026; pve-pbs time zone set to UTC)
| Job | Datastore | Schedule (UTC) | Settings |
|---|---|---|---|
| Node backups (Proxmox VE job "Nightly to pve-pbs") | local-store | 08:00 daily | all guests except 300, 301, 302, 400; snapshot mode |
| prune-local | local-store | 09:30 daily | keep 7 daily, 4 weekly, 3 monthly |
| GC | local-store | 09:45 daily | |
| sync-local-to-b2 | local-store to b2-store | 10:00 daily | remove-vanished false (B2 keeps its own longer history); encrypted-only 1 (confirmed Nov 16, 2026), so unencrypted backups never reach B2 |
| prune-b2 | b2-store | 12:00 daily | keep 7 daily, 4 weekly, 6 monthly |
| GC | b2-store | sun 12:30 | |
| verify-local | local-store | sun 14:00 | ignore verified, re-verify after 30 days |
| verify-b2 | b2-store | 1st of month 15:00 | ignore verified, re-verify after 90 days |

## Access from the cluster
- User `pve-backup@pbs`, token `pve-backup@pbs!pve-cluster` (secret in Vaultwarden): DatastoreBackup on `/datastore/local-store`, DatastoreReader (read only) on `/datastore/b2-store` for offsite restores.
- Proxmox VE storage `pve-pbs` (cluster-wide): server 10.0.60.x, datastore local-store, fingerprint `08:1f:f5:0f:de:f7:f9:3f:70:90:ad:b2:78:32:21:ae:d3:fc:da:af:c0:0b:96:bd:c0:3d:66:b5:69:f6:33:50`, `prune-backups keep-all=1` (retention is done on pve-pbs). Active Nov 16, 2026.
- Proxmox VE storage `pve-pbs-b2` (cluster-wide, added Nov 16, 2026): same server, token and fingerprint, datastore b2-store, read only. Restore path from the offsite copy; no backup job uses it. Its key is a copy of the same encryption key at `/etc/pve/priv/storage/pve-pbs-b2.enc`.
- **Client-side encryption:** key auto-generated at `/etc/pve/priv/storage/pve-pbs.enc` (no passphrase). Copy saved in Vaultwarden (`pve-pbs-encryption-key`) plus an offline copy. Without it no backup can be restored.

## Open
- Disaster-recovery drill: a fresh Proxmox Backup Server reconnecting to the B2 bucket (`--reuse-datastore`) and restoring, to prove recovery without pve-pbs or its cache
- Verify jobs: run against local-store. Verifying the B2 datastore downloads every chunk, so keep it monthly or less.

## Hardening (Nov 16, 2026, script `/root/harden-pve-pbs.sh`, based on [[sop-harden-debian-vps]])
- User `admin` (sudo; password in Vaultwarden), key from the workstation (`id_ed25519.pub`) in both `/root` and `/home/admin` `authorized_keys`.
- `/etc/ssh/sshd_config.d/00-hardening.conf`: root login off, password and keyboard-interactive off, `AllowUsers admin`. Verified with `sshd -T`.
- nftables host firewall: input policy drop, allows loopback, established, ICMP, TCP 22 and 8007. Service enabled for boot.
- unattended-upgrades for Debian security updates only; Proxmox packages stay manual.
- fail2ban `sshd` jail, systemd backend.
- Web UI login `root@pam` still works (not SSH).
- Confirmed from the workstation Nov 16, 2026: `ssh root@10.0.60.x` refused (Permission denied), `ssh admin@10.0.60.x` logs in. nftables service active, fail2ban sshd jail running.

## Tests
- Nov 16, 2026: manual backup of [[dev-git]] (CT 3300) to local-store, encrypted (key fingerprint `1e:08:55:f2:92:43:22:23`), then restored from pve-pbs to throwaway CT 9300 on pve-env1 (1.406 GiB in 1m 15s, about 19 MiB/s). Config came back intact. Restore works.
- Nov 16, 2026: manual run of `sync-local-to-b2` copied that backup to b2-store: 717.793 MiB in 351 chunks, about 6.1 MiB/s, TASK OK. A second run found no new data, so only changes upload.
- Nov 16, 2026: restored the same dev-git backup from **b2-store** (storage `pve-pbs-b2`) to throwaway CT 9300, decrypted with the same key, 1.406 GiB in 1m 16s, config intact, copy destroyed. Some chunks may have come from pve-pbs's local cache rather than Backblaze, so this proves the B2 datastore, access and decryption, not recovery without pve-pbs.

---
name: incident-2026-10-15-pve-services-nvme-dropout
type: incident
date: 2026-10-15
status: resolved
---

# Incident: pve-services NVMe Stopped Responding, All Volumes Remounted Read-Only

**Date:** 2026-10-15 ~14:01 UTC (fault) / 2026-10-17 (found and fixed)
**Systems affected:** [[pve-services]] and every guest on it: [[pve-ca-root]] (500), [[pve-ca-intermediate]] (501), [[pve-int-stepca]] (511), [[authentik]] (2201), [[pve-bastion]] (2202), [[pve-authtunnel]] (2100), [[ubuntu-401]] (401)
**Status:** Resolved. Root cause probable, not confirmed (see Root Cause).

## Summary

The system NVMe in pve-services (Samsung 990 PRO 1TB) stopped answering I/O at about 14:01 UTC on Oct 15. Every LVM volume on it hit I/O errors and remounted read-only. The node stayed pingable, but SSH closed connections before the banner, the web UI timed out, the Proxmox cluster showed all of its guests with grey "?" status, and Authentik returned "Server Error". Nobody noticed for about two days. A power cycle brought the drive back with clean SMART data. Drive power saving (APST) was disabled on Oct 17.

## Timeline (UTC)

- **Oct 10 ~07:50** pve-services boots. Stays up from here.
- **Oct 15 14:00:41** Load average 0.09. Normal.
- **Oct 15 14:01:11** 1-minute load 2.59. Last journal entry from pve-services. Journal sequence numbers 2714686 to 2714688 are consecutive, so nothing else (no logins, no kernel messages) was logged in the final minute.
- **Oct 15 ~14:03** Console: `EXT4-fs ... error 5` (I/O error) on dm-9, then aborted journals and read-only remounts on dm-8 (dockerd, likely LXC 511), dm-9, dm-6 (qmgr), and dm-7 (systemd-journal, kmmpd).
- **Oct 15 to Oct 17** Node dark. No alert fired.
- **Oct 17 ~14:12** Investigation starts (SSH `kex_exchange_identification: Connection closed by remote host`, web UI `ERR_TIMED_OUT`).
- **Oct 17 ~14:38** Console photographed, node power cycled.
- **Oct 17 14:44 to 15:05** One-off vzdump backups of 500, 501, 511, 2100, 2201, 2202 streamed to pve-env1.
- **Oct 17 ~15:14** APST disabled via kernel parameters, node rebooted, verified.

## Investigation

| Check | Result |
|-|-|
| Ping 10.0.0.x | Replies. Kernel network stack alive. |
| `ssh -vvv admin@10.0.0.x` | `kex_exchange_identification: Connection closed by remote host`. sshd accepted and dropped before sending its banner. |
| Web UI :8006 | `ERR_TIMED_OUT`. |
| Proxmox Summary graphs (week) | CPU ~2%, RAM ~6 of 16 GiB, IO delay and IO/memory pressure flat. Every graph stops at the same moment with no ramp. Rules out resource exhaustion. |
| Elastic `proxmox-logs` | Last event 14:01:11 UTC Oct 15. Filebeat uptime 5.5 days, so no reboot beforehand. |
| Elastic search for SSH, sudo, session events 07:00 to 10:00 CDT Oct 15 | Only hourly cron (`:17`). No interactive logins. |
| Physical console | EXT4 I/O errors and read-only remounts on four dm devices within ~3 minutes. |
| `smartctl -a /dev/nvme0` (after reboot) | PASSED. Critical Warning 0x00, Media and Data Integrity Errors 0, Error Information Log Entries 0, Percentage Used 1%, Available Spare 100%, 4,310 power-on hours, 7 unsafe shutdowns, 52°C. Firmware 5B2QJXD7. |
| `journalctl -k -b -1 \| grep nvme` | Only normal boot lines from Oct 10. The fault was never written because the journal is on the same drive. |
| `nvme get-feature -f 0x0c -H` | APST enabled. PS3 after 100 ms idle, PS4 (deepest) after 2000 ms idle. |

## Compromise assessment

Considered and ruled out. Four volumes failing at once is a storage fault, not attacker activity. No CPU or network spike before the cutoff, and the final minute of journal shows no logins or privilege use.

## Root Cause

Proximate: the NVMe stopped responding to the host, so every volume on it failed and remounted read-only.

Probable underlying cause: the Samsung 990 PRO dropping off the PCIe bus while entering or leaving a low-power APST state. The drive's own error log and SMART counters are clean, which fits the host losing contact with the drive rather than the flash failing. This matches widely reported 990 PRO behavior on Linux, where the kernel itself suggests `nvme_core.default_ps_max_latency_us=0 pcie_aspm=off`.

Not confirmed: the `nvme` kernel messages from the fault (for example `controller is down; will reset`) were never captured. They scrolled off the console and could not be written to the journal.

Power-on hours put the drive's install around mid-April 2026, so this is the same drive as [[incident-2026-05-30-pve-services-pmxcfs]]. That incident's disk I/O error is very likely the same failure mode.

## Fix

Added to `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub` (pve-services boots through plain GRUB, `proxmox-boot-tool` is not in use). Backup of the original at `/etc/default/grub.bak`.

```
nvme_core.default_ps_max_latency_us=0 pcie_aspm=off
```

Then `update-grub` and reboot. Verified Oct 17: `/proc/cmdline` contains both, and `nvme get-feature /dev/nvme0 -f 0x0c -H` reports `APSTE: Disabled`.

## Backups taken

There were no backups of any pve-services guest before this. One-off vzdump archives, all verified with `zstd -t`, are in admin's home directory on [[pve-env1]] (on pve-env1's root filesystem):

- `vzdump-qemu-500-2026_10_17.vma.zst`
- `vzdump-qemu-501-2026_10_17.vma.zst`
- `vzdump-lxc-511-2026_10_17.tar.zst`
- `vzdump-lxc-2100-2026_10_17.tar.zst`
- `vzdump-lxc-2201-2026_10_17.tar.zst`
- `vzdump-lxc-2202-2026_10_17.tar.zst`

401 was skipped on purpose (throwaway). Method: `sudo vzdump <id> -mode snapshot -compress zstd -stdout 1 | ssh admin@10.0.0.x "cat > <file>"` from a `ssh -A` session, because admin on pve-services has no key trusted by pve-env1 and the cluster has no shared backup storage.

## Open items

- Node was down about two days before anyone noticed. Same gap as the May 30 incident. No alert exists for a node going silent.
- No recurring backups of pve-services guests. The archives above are one-off and sit on pve-env1's root filesystem.
- Root CA key exists only inside VM 500 and its backup. No plain offline copy yet.
- Check whether firmware newer than 5B2QJXD7 exists for the 990 PRO.
- Guests that remounted read-only should be checked for filesystem damage after the reboot.
- If the drive drops out again with APST off, treat it as a hardware problem (drive, M.2 slot, or board) and replace.

## Related

- [[incident-2026-05-30-pve-services-pmxcfs]]
- [[pve-services]]
- [[proxmox-troubleshoot-cluster-quorum|Troubleshoot Cluster Quorum]]

# Incident: pve-env1 Silent Power-Off, Quorum Loss and NPM Outage

**Date:** 2026-05-27 to 2026-05-28
**Systems affected:** pve-env1 (HP EliteDesk 800 G5 Mini, 10.0.0.12), Nginx Proxy Manager, internal domain routing, cluster quorum
**Status:** Recovered. Root cause not confirmed.

## Summary

Three days after the May 24 power outage, pve-env1 went dark on its own while idle, with no power event affecting the other nodes on the same strip. Quorum broke, NPM went down with it, and internal domain routing stopped working. The node was recovered with a reboot. Every software and thermal cause that could be checked from logs was ruled out. The leading suspect is the node's external power brick, possibly stressed by the May 24 outage, but this has not been confirmed.

## Timeline

Times are from pve-env1's clock, which was set to Hawaii time (HDT). The node reported HDT while the lab is in Central time.

- **Apr 20 20:42** Boot -2 starts. Node runs cleanly for 37 days.
- **May 24 13:15** All corosync links drop at once during the household power outage. Cluster self-recovers by 13:17. pve-env1 itself stays up.
- **May 27 00:10** Routine pveproxy restart. Nothing else of note.
- **May 27 08:04:47** Boot -2 ends. Last log line is a routine Filebeat metric. CPU load near idle.
- **May 27 16:05** Boot -1 starts (recovery). About 8 hours down.
- **May 27 20:56** Boot -1 ends.
- **May 28 05:28** Boot 0 starts (recovery after quorum, NPM and domain routing were found broken that morning).

## Investigation

| Check | Result |
|-|-|
| `journalctl -b -2` corosync | Only the May 24 outage and its recovery. Nothing at 08:04 on May 27. |
| Journal, 07:50 to 08:05 | Routine activity, then nothing. No shutdown sequence, panic, OOM, ACPI or thermal event. |
| Kernel log, boot -2 | Last kernel event was NIC recovery from May 24. Silent after that. |
| `mcelog` | No machine check errors. |
| `sensors` | Package 45°C at idle. Normal. |
| Power | Same strip and outlet as the other three nodes, which stayed up. |
| Power bricks | None on any node warm to the touch. |

A failure that leaves no trace at all, with the node idle and other nodes on the same power unaffected, points at something below the OS: the external power brick or its connections, or a firmware-level power event.

## Root Cause

Not confirmed. Working hypothesis is an intermittent fault in pve-env1's external power brick, possibly from the hard power cut on May 24. A scheduled power-off in the BIOS has not been ruled out either.

## Recovery

Rebooted pve-env1. Quorum, NPM and domain routing came back.

## Open items

- Check the HP BIOS event log (F10 at POST) for a power or hardware event around May 27 08:04 node time, and check for any scheduled power-off setting.
- The boot list shows a **second** gap: boot -1 ended May 27 20:56 and boot 0 started May 28 05:28. Only the first gap was investigated. Boot -1's logs have not been checked for how it ended.
- If it happens again, record the time to look for a pattern. Replace the power brick on a third occurrence.
- Fix node timezones (set to Hawaii time instead of Central).
- Confirm `onboot` is set on pve-env1's VMs (200, 290).

## Related

- [incident-review-may24-power-onboot.md](incident-review-may24-power-onboot.md)

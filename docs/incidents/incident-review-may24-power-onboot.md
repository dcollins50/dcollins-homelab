# Incident: Power Outage, SOC VMs Did Not Restart (onboot Not Set)

**Date:** 2026-05-24 (outage) to 2026-05-26 (found and fixed)
**Systems affected:** Entire cluster (power loss). soc-stack (VM 600) and wazuh-manager (VM 601) on pve-env2 stayed down.
**Status:** Resolved

## Summary

A household power outage took down every node. All nodes came back on their own, but soc-stack and wazuh-manager did not start because neither VM had `onboot` set. The SOC stack was down for roughly a day and a half until the VMs were found stopped and started by hand.

## Timeline

Node clocks were set to Hawaii time, which made the host logs look like two separate events at first. Converting to UTC showed one event.

- **May 24 ~22:14-22:15 UTC** Power lost. Both VMs' journals stop mid-operation with no shutdown sequence. pve-env2's previous boot ends at the same moment.
- **May 24** Power returns. All nodes boot. Proxmox runs `startall`, which skips VMs without `onboot`.
- **May 24 to 26** soc-stack and wazuh-manager sit stopped. Filebeat on pve-env2 fills its queue (3200 events, 100%) trying to reach Logstash.
- **May 26** VMs found stopped with the node up. Started manually. Filebeat immediately flushed about 6400 backed-up events.
- **May 26** Checked for other causes: no `qmstop` or `qmshutdown` tasks in the Proxmox task log, no OOM events in `dmesg`, 32 GB RAM on pve-env2 with 12 GB available. `onboot` confirmed missing on both VMs. Power outage confirmed.

## Root Cause

There is no UPS, so a power outage takes the whole cluster down hard. The nodes recover by themselves, but VMs only auto-start if `onboot: 1` is set in their config. VMs 600 and 601 did not have it.

## Fix

```bash
sudo qm set 600 --onboot 1
sudo qm set 601 --onboot 1
sudo qm config 600 | grep onboot
sudo qm config 601 | grep onboot
```

## Lessons Learned

- Every VM that is supposed to be running should have `onboot: 1`. Check this whenever a VM is created or cloned.
- In the logs, a hard power cut looks like a journal that just stops, with no shutdown sequence and no panic.
- Node timezone settings need fixing before the next timeline reconstruction.

## Open items

- `onboot` audit across all four nodes. On May 28 it was unclear whether this had been done for the VMs on pve-env1 (200, 290).
- No UPS. Deferred.
- Node timezones set to Hawaii time instead of Central.

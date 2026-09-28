---
name: standardize-node-clocks-utc
type: decision
status: live
---

## Decision

Standardize every machine in the homelab to UTC: all 4 Proxmox hosts, every VM and LXC guest, OPNSense, and Heimdall.

**Status as of Sep 25, 2026:** 16 hosts confirmed UTC — all 4 Proxmox hosts, Heimdall, 5 VMs (pve-iris, soc-stack, services-host, pve-ca-intermediate, soar-host), and 6 LXCs (authentik, pve-int-stepca, pve-bastion, pve-authtunnel, npm-dmz). wazuh-manager was already UTC beforehand. See [[session-2026-09-25-utc-rollout]] for the full run.

**Still outstanding:** kali-attack, ubuntu-401, pve-ca-root (were stopped during the SSH key rollout, never got a key or the UTC change); OPNSense (needs the web UI, not scriptable the same way); metasploitable2 and DVWA (deliberately excluded, stay on password auth); malware-win11 (Windows, air-gapped); docker-host-template (stopped, needs booting or offline edit).

## Why

Discovered during [[session-2026-09-25-rule-100004-confirmation]] while confirming Wazuh suppression rule 100004 across all four nodes. Checking each node's clock turned up three different timezones on machines that were assumed to be consistent:

- pve-env1: HDT
- pve-env2: HDT
- pve-gateway: CDT
- pve-services: CDT
- wazuh-manager (VM): UTC, apparently by accident of provisioning rather than any fleet-wide standard

No other VM, LXC, OPNSense, or Heimdall clock has been checked yet, so the actual spread across the fleet is unknown beyond these five.

This is the third time a node's local timezone has had to be manually converted to UTC to line up a local log timestamp with a Wazuh/Kibana query, most recently during the Sep 24 alert engineering session and again during the Sep 25 rule 100004 confirmation. Manual conversion is slow and error-prone, especially with pve-env1 on HDT specifically (a 6-hour offset from CDT, easy to get wrong under time pressure).

## What was ruled out

Documenting each node's individual offset instead of standardizing (e.g., a reference table kept in the vault). Rejected because it still requires a manual conversion step every time a timestamp needs to be checked against Kibana/Elasticsearch, which is UTC-native. Standardizing removes the conversion step entirely instead of just making it faster to look up.

## Implementation notes (for whoever picks this up)

- Proxmox hosts, VMs, and LXCs (all Debian/systemd-based so far): `sudo timedatectl set-timezone UTC`, takes effect immediately, no reboot needed. Scriptable as a simple SSH loop across the fleet once the full machine list is known.
- OPNSense is FreeBSD-based; `timedatectl` does not apply. Timezone is set via System > Settings > General in the web UI. Worth checking whether OPNSense's NTP configuration also needs adjustment at the same time.
- Heimdall (Raspberry Pi, Debian-based) uses the same `timedatectl` approach as the Proxmox fleet.
- Full inventory of VMs/LXCs to include hasn't been built yet; the [[homelab-soc]] VM inventory list is the starting point.

## Related
- [[session-2026-09-25-rule-100004-confirmation]]
- [[session-2026-09-24-alert-engineering-baseline]]

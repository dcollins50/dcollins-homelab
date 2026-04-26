# HOMELAB SOC -- Phase 2 Complete
**Date:** April 25, 2026
**Author:** Daniel Collins

---

## Overview

Phase 2 noise reduction and rule tuning is complete. All 20 rules from the Phase 1 baseline list received documented tuning decisions. Custom Wazuh rules were written, validated with xmllint, and deployed to wazuh-manager. Alert volume is expected to drop significantly over the next 7 days as suppression rules take effect.

---

## Infrastructure Actions Taken

### Disk Expansion -- wazuh-manager (VM 601)

The root partition was at 100% capacity. The LVM logical volume was expanded from 23.5G to 47G using unallocated space already present on the 50G disk. No data loss. Primary disk consumer is the Wazuh CVE vulnerability database at `/var/ossec/queue/vd/feed/`.

### Kernel Patches -- CVE Remediation

Three hosts had level 13 CVE alerts identified during Phase 1. Patches were applied where available:

| Host | Status |
|---|---|
| services-host (VM 200) | Patched and rebooted |
| soc-stack (VM 600) | Patched and rebooted |
| wazuh-manager (VM 601) | Patched |
| VM 401 (ubuntu, pve-services) | Patch unavailable upstream -- monitoring |

### Unintended Services Discovered and Remediated -- VM 401

Alert investigation during Phase 2 revealed two services running on VM 401 that were not intentionally configured as persistent services. Both were disabled. Services remain installed and can be started manually when needed.

---

## Phase 2 Results

**20 of 20** baseline rules received documented tuning decisions. Tuning decisions and the custom rules file are kept in internal documentation only and are not published here.

**Suppression rules written and deployed:** 9 rules

**Rules accepted as-is:** 11 rules

All custom rules passed xmllint validation. Wazuh manager restarted cleanly after each change.

---

## Notable Findings

**Rootcheck false positive on system binaries:** Wazuh flagged several standard PAM binaries as trojaned on two Proxmox nodes. Investigated with `dpkg --verify` -- binaries confirmed clean. False positive caused by Wazuh generic regex signatures matching strings in legitimate PAM packages. Documented in tuning log.

**Unidentified systemd transient unit failures:** Regular failures detected every 6 hours on two hosts from ephemeral systemd units. Source not yet identified -- journal logs did not capture sufficient detail. Flagged as open item for Phase 3 investigation.

**CIS benchmark oscillation:** GSSAPIAuthentication compliance check oscillating between passed and failed on multiple agents despite the setting being correctly configured in the running sshd config. Likely caused by package updates temporarily modifying sshd_config. Flagged for Phase 3 investigation.

---

## Open Items

| Item | Priority |
|---|---|
| VM 401 kernel patch | High -- monitor Ubuntu for upstream fix |
| Systemd transient unit failures | Medium -- identify source process |
| CIS benchmark oscillation | Medium -- identify which agents and stabilize sshd_config |
| Phase 2 volume verification | Low -- recheck alert counts around May 2 |

---

## Phase 2 Verification Checklist

| Checkpoint | Status |
|---|---|
| All 20 baseline rules have a documented tuning decision | PASS |
| Custom rules file passes validation | PASS |
| Wazuh manager restarted successfully after each rule change | PASS |
| Alert volume measurably reduced from Phase 1 baseline | PENDING -- recheck May 2 |
| No level 12 or above rules suppressed without documented investigation | PASS |
| Tuning decision log saved with entry for each changed rule | PASS |

---

## Next Steps

Phase 3 is alert triage -- a recurring daily and weekly practice built on the cleaned-up alert stream from Phase 2. Waiting approximately 3-4 days before starting regular triage sessions to allow suppression rules to clear the backlog. Phase 4 is DFIR-IRIS case management, supporting Phase 3 with formal case tracking for true positives and escalations.

---

*github.com/dcollins50/dcollins-homelab | Internal Documentation*

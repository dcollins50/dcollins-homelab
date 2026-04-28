# HOMELAB SOC -- Phase 2.5: Midweek Baseline State Report
**Date:** April 28, 2026
**Days since Phase 2 tuning:** 3 (tuning completed April 25, 2026)
**Author:** Daniel Collins

---

## Overview

This document records the observed state of the Wazuh SIEM three days after Phase 2 noise reduction and rule tuning was completed. The 7-day alert count still reflects a mixed window of pre- and post-tuning data. The 2-day window is used as the primary signal for post-tuning assessment.

---

## Alert Volume

| Window | Count | Notes |
|---|---|---|
| Last 7 days | 17,426 | Mixed pre/post-tuning window -- not a clean comparison |
| Last 2 days | 429 | Post-tuning only -- primary signal |
| Phase 1 baseline (7-day) | ~23,000+ | Dominated by rule 87907 Docker healthcheck noise |

The 2-day post-tuning rate extrapolates to roughly 1,500 alerts per week, representing a reduction of approximately 93% from Phase 1 baseline. The 7-day count remains elevated because it captures four days of pre-tuning data and will normalize by approximately May 2, 2026.

---

## Suppression Verification

Rule 87907 (Docker healthcheck exec_start events) confirmed at zero alerts in the last 2 days:

```
"count" : 0
```

Suppression is functioning as intended.

---

## Alert Distribution by Rule Group (Last 2 Days)

| Rule Group | Count | Assessment |
|---|---|---|
| ossec | 329 | Expected -- base OSSEC framework events |
| syscheck | 229 | Expected -- FIM activity across cluster |
| syscheck_file | 229 | Expected -- file-level FIM |
| syscheck_entry_modified | 177 | Expected -- primarily Proxmox cluster state files |
| rootcheck | 100 | Expected -- periodic system policy checks |
| syslog | 67 | Expected -- standard system log events |
| pam | 45 | Expected -- authentication framework |
| authentication_success | 32 | Present -- no suppression impact |
| syscheck_entry_added | 30 | Expected -- file additions |
| syscheck_entry_deleted | 22 | Expected -- file deletions |
| local | 15 | Expected -- custom local rules |
| systemd | 15 | Expected -- service lifecycle events |
| sudo | 12 | Expected -- privilege escalation logging |
| sshd | 10 | Present -- SSH activity monitored |
| vulnerability-detector | 10 | Present -- CVE detection active |
| docker | 8 | Reduced from 13,544 (7-day pre-tuning) -- suppression working |
| docker-error | 8 | Residual -- rule 86003, non-healthcheck Docker events |
| authentication_failed | 2 | Present -- no suppression impact |
| invalid_login | 2 | Present -- no suppression impact |

All high-confidence signal categories are present and active. No detection coverage gaps identified.

---

## Over-Normalization Assessment

The following categories were verified as still active post-tuning:

| Category | Status |
|---|---|
| authentication_success / authentication_failed | Active |
| syscheck / FIM | Active |
| vulnerability-detector | Active |
| rootcheck | Active |
| sshd | Active |
| sudo | Active |

Tuning has not degraded detection coverage. Noise reduction is isolated to low-value Docker healthcheck telemetry. High-confidence security signal categories are unaffected.

---

## Alert Volume by Agent (Last 2 Days)

| Agent | Count | Notes |
|---|---|---|
| pve-env1.homelab.local | 121 | Top agent -- Proxmox FIM on /etc/pve/ cluster state files |
| pve-gateway | 75 | Normal cluster activity |
| pve-services | 75 | Normal cluster activity |
| pve-env2.homelab.local | 69 | Normal cluster activity |
| wazuh-manager | 40 | Expected -- self-monitoring |
| soc-stack | 29 | Expected |
| services-host | 16 | Substantially reduced from Phase 1 -- Docker suppression working |
| jetson | 4 | Low volume, expected |

Notable: services-host dropped from the highest-volume agent in Phase 1 to near the bottom, confirming rule 87907 suppression is the primary driver of overall volume reduction.

Notable: VM 401 (ubuntu, pve-services, 10.0.30.40) did not appear in the last 2 days of agent activity. VM is powered off -- expected.

---

## Pipeline Health

| Index | Documents | Status |
|---|---|---|
| wazuh-alerts-4.x-2026.04.26 | 233 | Green |
| wazuh-alerts-4.x-2026.04.27 | 244 | Green |
| wazuh-alerts-4.x-2026.04.28 | 142 (partial day) | Green |

All indices healthy. Ingestion pipeline operating normally.

---

## Open Items

| Item | Priority | Notes |
|---|---|---|
| pve-env1 FIM volume | Low | 121 alerts in 2 days -- Proxmox cluster state files identified as source. Middle-path suppression planned for .rrd, .clusterlog, and lrm_status paths. |
| 7-day window normalization | Informational | Recheck May 2 for a clean post-tuning 7-day baseline |

---

## Interpretation

Alert volume is down approximately 93% from Phase 1 baseline on a post-tuning 2-day window. Noise reduction is isolated to Docker healthcheck telemetry as intended. All high-confidence detection categories remain active including authentication, FIM, rootcheck, SSH, sudo, and vulnerability detection. The SIEM is operating in a cleaner signal state without meaningful loss of coverage. The primary remaining noise source is Proxmox cluster state FIM on pve-env1, which has a documented suppression plan pending implementation.

---

*github.com/dcollins50/dcollins-homelab | Internal Documentation*

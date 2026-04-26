# HOMELAB SOC -- Phase 1 Baseline Report
**Date:** April 25, 2026
**Author:** Daniel Collins
**Classification:** Public

---

## Overview

Phase 1 establishes the SIEM baseline -- a 7-day snapshot of alert volume, rule distribution, and severity breakdown across all enrolled Wazuh agents. This baseline serves as the reference point for Phase 2 tuning decisions and ongoing anomaly detection.

---

## Environment

| Field | Value |
|---|---|
| Baseline Window | April 18 to April 25, 2026 (7 days) |
| Total Alerts (all-time) | 31,177 |
| Agents Enrolled | 8 active agents |
| Wazuh Version | 4.14.4 |
| ELK Version | 8.19.12 |

**Enrolled agents:** pve-gateway, pve-services, pve-env1, pve-env2, services-host, soc-stack, wazuh-manager, Jetson (arm64)

---

## Alert Volume by Day

| Date | Alert Count | Notes |
|---|---|---|
| April 16 | 2,421 | |
| April 17 | 2,483 | |
| April 18 | 1,334 | |
| April 19 | 3,091 | Agent rollout completed |
| April 20 | 277 | Anomaly -- under investigation |
| April 21 | 2,369 | |
| April 22 | 4,922 | Highest daily volume |
| April 23 | 3,246 | |
| April 24 | 3,170 | |
| April 25 | 2,703 | Baseline collection date |

April 20 showed anomalously low volume compared to surrounding days. Under investigation.

---

## Top Alert Sources

Alert volume was dominated by a small number of high-frequency rules. The top categories were:

- Docker container event monitoring (services-host, Docker wodle)
- Vulnerability detection scan results (active CVEs and resolved CVEs)
- File integrity monitoring (FIM) events across Proxmox nodes
- Host-based anomaly detection (rootcheck) across all agents
- Package management events (dpkg installs, removals, half-configured states)
- Systemd service lifecycle events
- Authentication events (PAM, Proxmox)
- CIS benchmark compliance checks

Detailed rule rankings and per-rule analysis are maintained in internal documentation.

---

## Severity Distribution

The overwhelming majority of alerts during the baseline period were informational (level 3), which is expected for a stable, non-targeted homelab environment. A small percentage of alerts were medium severity (levels 5-10), and a very small number were high severity (level 13).

High severity alerts all originated from the Wazuh vulnerability detector identifying unpatched CVEs on specific kernel versions. No alerts indicated active compromise or unauthorized access.

---

## High Severity Findings

11 level-13 alerts were identified across 3 agents, all from the Wazuh vulnerability detector (rule 23506). These alerts identified active unpatched CVEs on Linux kernel packages.

All three affected hosts were investigated. Patches were applied where available upstream. One host remained unpatched pending an upstream kernel fix from Ubuntu.

Full CVE details and affected host information are maintained in internal documentation only.

---

## Agent Alert Distribution

`services-host` was the highest-volume agent by a significant margin, driven by Docker container monitoring events. Proxmox cluster nodes generated moderate volume primarily from file integrity monitoring on cluster state files. The soc-stack and wazuh-manager generated lower volumes consistent with their roles.

---

## SIEM Baseline Dashboard

A dedicated Kibana dashboard was built on the `wazuh-alerts-4.x-*` index with 6 panels:

- Total Alerts (7 Days) -- Metric
- Alert Volume Over Time -- Bar chart
- Top 20 Rules by Alert Count -- Table
- Alert Volume by Agent -- Horizontal bar chart
- Alert Distribution by Severity Level -- Pie chart
- Alert Volume by Rule Group -- Bar chart

---

## Phase 1 Verification Checklist

| Checkpoint | Status |
|---|---|
| SIEM Baseline dashboard saved in Kibana with all 6 panels populated | PASS |
| Top 20 rules ranked by alert count with descriptions recorded | PASS |
| Each agent's alert contribution identified | PASS |
| All level 12 and above alerts identified and flagged for review | PASS |
| Baseline document saved with initial assessment column completed | PASS |

---

## Next Steps

Phase 2 applies tuning decisions to all 20 baseline rules to reduce noise and improve signal quality. Phase 2 is documented separately.

---

*github.com/dcollins50/dcollins-homelab | Internal Documentation*

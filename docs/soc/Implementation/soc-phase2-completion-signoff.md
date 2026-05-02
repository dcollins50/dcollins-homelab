# HOMELAB SOC - Phase 2: Completion Sign-Off

**Date:** May 2, 2026
**Author:** Daniel Collins
**Phase:** 2 of 5

---

## Summary

Phase 2 (SIEM Tuning and Noise Reduction) is complete. All tuning decisions have been implemented, verified, and documented. The 7-day post-tuning baseline has been established as of May 2, 2026.

---

## Final Baseline Numbers

| Metric | Value |
| --- | --- |
| Phase 1 baseline (7-day) | ~23,000+ alerts |
| Phase 2 clean 7-day baseline | 4,253 alerts |
| Overall reduction | ~82% |
| Post-tuning 2-day reduction (vs Phase 1 rate) | ~93% |
| Top rule (rule 550, FIM) - 7-day | 671 |
| Top rule (rule 550, FIM) - 2-day post-suppression | 243 |

---

## Tuning Decisions Applied

| Rule | Decision | Rationale | Date |
| --- | --- | --- | --- |
| 87907 - Docker healthcheck exec\_start | Suppress (global) | ~13,000 alerts in 7 days, zero security value. Docker healthcheck telemetry. | April 25, 2026 |
| 550 - FIM on /etc/pve/ | Suppress (path-specific) | 618 alerts in 7 days from Proxmox cluster state files (.rrd, .version, lrm\_status, authkey). Normal cluster operation. | April 25, 2026 |

All suppression decisions are documented inline in `/var/ossec/etc/rules/local_rules.xml` on wazuh-manager with date, volume, and rationale.

---

## Detection Coverage Verification

All high-confidence security signal categories confirmed active post-tuning:

| Category | Status |
| --- | --- |
| FIM (syscheck) - non-pve paths | Active |
| Vulnerability detection | Active |
| Rootcheck | Active |
| SSH monitoring | Active |
| Sudo privilege escalation | Active |
| Authentication success/failure | Active |
| Docker (non-healthcheck events) | Active |
| PAM | Active |

No over-normalization. Noise reduction is isolated to low-value telemetry.

---

## Infrastructure Changes During Phase 2

| Change | Status |
| --- | --- |
| Wazuh agent rollout - all active nodes and VMs | Complete |
| VM 700 agent | Deferred (VM shut down) |
| 7 official Wazuh dashboards imported and populated | Complete |
| Vulnerability detection via Elasticsearch indexer connector | Active |
| Docker wodle enabled on services-host | Active |
| Logstash ssl\_verification\_mode set to "full" | Complete (May 2, 2026) |
| Intermediate CA cert deployed to soc-stack | Complete (May 2, 2026) |

---

## Agent Coverage

| Agent | Status |
| --- | --- |
| soc-stack (VM 600) | Active |
| wazuh-manager (VM 601) | Active |
| services-host (VM 200) | Active |
| pve-gateway | Active |
| pve-services | Active |
| pve-env1 | Active |
| pve-env2 | Active |
| Jetson Orin Nano | Active |
| VM 401 ubuntu (pve-services) | Agent installed, VM currently powered off |
| VM 700 services-host2 | Deferred - VM shut down |

---

## Open Items Carried Into Phase 3

| Item | Priority | Notes |
| --- | --- | --- |
| Root password SSH disabled on Proxmox nodes | High | Still OPEN - SOP-SEC-001 Section 1 |
| Suricata IPS mode | Medium | Suricata running IDS mode on OPNSense. IPS mode transition is a Phase 3+ item |
| Logstash ssl\_verification\_mode (suricata.conf) | Closed | Resolved May 2, 2026 |
| VM 700 Wazuh agent | Deferred | VM shut down, revisit when VM is reactivated |
| Standard-PC-Q35-ICH9-2009 hostname cleanup | Low | hostnamectl rename pending |
| OPNSense agentless SSH timeout | Low | Needs resolution |

---

## Related Documentation

- [Phase 1 Baseline - SOP and Learning Exercise](../soc-phase1-baseline.md)
- [Phase 2 Tuning Log](soc-phase2-tuning.md)
- [Phase 2.5 Midweek State Report](soc-phase2.5-midweek-state.md)
- [Phase 2.5 May 2 Close-Out](soc-phase2.5-may2-closeout.md)
- [Runbook: Logstash TLS Hardening](../../runbooks/runbook-logstash-tls-hardening.md)
- [SOP-SEC-001 Security Hardening](../../sop/sop-sec-001.md)

---

**Phase 2 is closed. Proceeding to Phase 3.**

---

*github.com/dcollins50/dcollins-homelab | Internal Documentation*

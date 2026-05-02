# HOMELAB SOC - Phase 2.5: May 2 Recheck and Close-Out

**Date:** May 2, 2026
**Author:** Daniel Collins
**Closes:** Open item from soc-phase2.5-midweek-state.md (April 28, 2026)

---

## Purpose

This document closes out the two open items identified in the April 28 midweek state report:

1. 7-day window normalization - recheck May 2 for a clean post-tuning 7-day baseline
2. pve-env1 FIM volume - middle-path suppression planned for cluster state file paths

---

## Item 1: 7-Day Baseline Confirmed

The April 28 midweek report noted that the 7-day alert count (17,426) was a mixed pre- and post-tuning window and would not normalize until approximately May 2.

Today's 7-day count: **4,253 alerts**

This is the clean post-tuning 7-day baseline. The window now contains only post-tuning data. This figure is the official Phase 2 baseline for comparison against future phases.

| Metric | Value |
| --- | --- |
| Phase 1 baseline (7-day) | ~23,000+ |
| Phase 2 clean 7-day baseline | 4,253 |
| Reduction from Phase 1 | ~82% on 7-day window |
| Reduction on 2-day post-tuning window (April 28) | ~93% |

**Status: CLOSED**

---

## Item 2: pve-env1 FIM Suppression Verified

The midweek report flagged rule 550 (integrity checksum changed) firing at elevated volume from `/etc/pve/` Proxmox cluster state files and documented a middle-path suppression plan.

Verification confirmed the suppression was implemented on April 25, 2026:

**Rule (local_rules.xml on wazuh-manager):**

```xml
<!-- Rule 550: Suppress FIM alerts for /etc/pve/ on Proxmox nodes
     Decision: Suppress (path-specific) | Date: 2026-04-25 | Volume: 618 in 7 days
     Reason: /etc/pve/ contains Proxmox cluster state files (.rrd, .version,
             lrm_status, authkey) that change continuously during normal
             cluster operation. Not a security signal. -->
<rule id="100003" level="0">
  <if_sid>550</if_sid>
  <field name="syscheck.path">/etc/pve/</field>
  <description>FIM: Proxmox cluster state change suppressed (expected)</description>
</rule>
```

**Post-suppression rule 550 count (last 2 days as of May 2):** 243

The 243 remaining rule 550 alerts represent legitimate FIM activity from paths outside `/etc/pve/`. These are expected and healthy - the suppression is correctly scoped and has not over-normalized. Detection coverage is intact.

Note: The 7-day dashboard shows rule 550 at 671, which is inflated by pre-suppression data from April 25-26 before the rule took effect. The 2-day count of 243 is the accurate post-suppression signal.

**Status: CLOSED**

---

## Additional Work Completed on May 2

### Logstash TLS Certificate Verification

`ssl_verification_mode` was confirmed still set to `"none"` across all Logstash pipeline output blocks. This was resolved in the same session:

- Intermediate CA cert deployed to soc-stack at `/etc/logstash/certs/homelab-intermediate-ca.crt`
- All output blocks in `proxmox.conf` (4 blocks) and `suricata.conf` (1 block) updated to `ssl_verification_mode => "full"` with the CA cert path configured
- Hosts updated from `localhost:9200` to the Elasticsearch node IP to match the Elasticsearch certificate SANs
- Pipeline restarted and verified - documents actively writing, no certificate errors in logs

Full documentation: `docs/runbooks/runbook-logstash-tls-hardening.md`

---

## Phase 2 Summary

All Phase 2 tuning and verification work is complete. The SIEM is operating with a clean post-tuning baseline and all high-confidence detection categories verified active.

| Milestone | Status |
| --- | --- |
| Rule tuning complete | DONE |
| Docker healthcheck suppression (rule 87907) | DONE |
| pve-env1 FIM suppression (rule 100003) | DONE |
| All detection categories verified active | DONE |
| Logstash TLS verification | DONE |
| 7-day clean baseline established | DONE |
| Phase 2.5 open items closed | DONE |

**Phase 2 is closed. Ready to proceed to Phase 3.**

---

*github.com/dcollins50/dcollins-homelab | Internal Documentation*

---
name: sop-cluster-wake-health-check
type: sop
status: planned
---

# SOP: Cluster Wake / Health-Check / Update Automation

**Status:** Not designed yet. This file is a placeholder that captures the objective and known constraints, not a procedure — there is no build to document, and this file should not be treated as one until an actual approach is chosen.
**Objective:** Automate waking the Proxmox cluster from a shutdown state, health-checking it, and applying updates, rather than doing each step manually.

---

## Why this would be an SOP, not a runbook

Waking and validating the cluster touches every physical node, the switch, [[opnsense]], and whatever DNS is currently primary — none of that is one tool's job, and the point of automating it is specifically to not have to manually sequence across all of them.

## Why this matters

The Proxmox cluster has been shut down for extended periods before (1-2 months, as of Sep 2026) without a scripted way to bring it back up cleanly and confirm it's healthy. During that downtime, a separate house Pi became primary DNS for the personal workstation, with [[heimdall]] running as secondary — a manual workaround for the lack of automation here, not a permanent design.

## Known constraints going in

- Scripting skill is still developing — the stated plan is plain bash/SSH loops first, moving to Ansible later once the pattern is proven, not starting with Ansible.
- No decided scope yet for what "healthy" means to check for (quorum? all VMs at expected state? services responding?).
- No decided trigger — manual run, scheduled, or something else.

## Open design questions (none of these are answered yet)

1. **What counts as "wake the cluster"?** Power-on sequencing across four physical nodes, or just confirming they're already on and bringing services up?
2. **What does the health check actually verify?** At minimum this probably wants Corosync quorum (see [[proxmox-troubleshoot-cluster-quorum|Troubleshoot Cluster Quorum]] for what "unhealthy" looks like), but whether it also checks individual VM/service state hasn't been decided.
3. **Where does "apply updates" fit?** OPNSense already has its own considered update cadence (see [[opnsense-check-apply-updates|Check and Apply Updates]]) — deliberately holding back a release with a known unpatched CVE. Does cluster-wide automation need to respect that same logic for the Proxmox nodes themselves, or is package-level OS patching a separate, simpler concern?
4. **DNS handoff.** If the house Pi is genuinely still primary DNS for the workstation, does waking the cluster need to also revert that, or is DNS deliberately being left as a manual step?

## Suggested first step, if you want to start this

Given the stated preference for plain bash/SSH before Ansible: a health-check-only script (read-only, no state changes) would be the lowest-risk starting point — confirm `pvecm status` shows quorate and all four nodes reachable over SSH, before attempting anything that actually powers hardware on or off.

## Related Documentation

- [[Infrastructure]]
- [[heimdall]]
- [[proxmox-troubleshoot-cluster-quorum|Troubleshoot Cluster Quorum]]
- [[opnsense-check-apply-updates|Check and Apply Updates (OPNSense)]]
- [[pihole-troubleshoot-primary-dns-failover|Primary DNS Failover / Outage Contingency]]

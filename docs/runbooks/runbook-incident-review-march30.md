**HOMELAB INFRASTRUCTURE**

Incident Review & Lessons Learned

March 30, 2026  |  SOC DNS Resolution & Firewall Audit Gap

| **Date** | March 30, 2026 |
| --- | --- |
| **Author** | Daniel Collins |
| **Affected Systems** | soc-stack (10.0.10.10), OPNSense, Heimdall RPi5 |
| **Severity** | Low — no data loss, no security breach; operational gaps only |
| **Status** | Resolved |

# **Background**

During a routine SOC infrastructure session, the objective was to resolve three known pending items: fix soc-stack DNS (change resolver from 192.168.100.1 to 10.0.10.1), correct a stale static route on Heimdall, and begin Wazuh Manager deployment. While addressing the DNS change, two previously undetected firewall gaps were discovered and remediated.

# **Session Timeline**

## **Issue 1 — soc-stack DNS Resolution Failure**

soc-stack was configured to use Pi-hole (192.168.100.1) as its DNS resolver. This address is on the WAN-side network, which OPNSense blocks by default from internal VLANs. DNS queries from soc-stack were being dropped, generating consistent firewall block log entries.

The intended fix was to point soc-stack at 10.0.10.1 (the OPNSense SOC VLAN gateway), where Unbound DNS was already running. However, testing revealed that DNS queries to 10.0.10.1 also timed out. Root cause: no firewall rule existed on the SOC interface permitting DNS traffic destined for the firewall itself.

- Resolution: Added OPNSense firewall rule — SOC net → This Firewall → TCP/UDP port 53

- Resolution: Added Unbound Query Forwarding rule — homelab.local → 192.168.100.1 (Pi-hole)

- Resolution: Updated soc-stack /etc/netplan config, changed nameserver from 192.168.100.1 to 10.0.10.1

- Verified: dig @10.0.10.1 google.com resolves correctly from soc-stack

## **Issue 2 — SSH to Heimdall Blocked from Workstation**

While preparing to SSH into Heimdall to address the stale static route, SSH connections timed out from the workstation on both 192.168.100.1 (rack-side) and 192.168.1.2 (home network-side). Heimdall was fully operational — Pi-hole web UI was accessible, ping responded, and sshd was active and listening on all interfaces.

Root cause: The LAN firewall ruleset contained explicit rules for DNS and ICMP to 192.168.100.1, but no rule permitting SSH. This gap was introduced during a prior firewall hardening session where overly permissive rules were replaced with scoped explicit rules. SSH to Heimdall was never validated after that hardening work and had been silently broken since.

Tailscale (100.74.169.33) was used as an out-of-band access path to diagnose and resolve the issue without requiring physical access to the RPi5.

- Resolution: Added OPNSense LAN firewall rule — LAN net → 192.168.100.1 → TCP port 22

- Verified: SSH from workstation to Heimdall successful

# **Root Cause Analysis**

## **Missing SOC DNS Firewall Rule**

The SOC VLAN was built with outbound HTTP/HTTPS and specific host rules, but DNS to the firewall was never explicitly permitted. This is a configuration gap that should have been caught during initial VLAN buildout or during the subsequent firewall audit.

## **SSH to Heimdall Not Validated Post-Hardening**

During the firewall audit, broad permissive rules were replaced with scoped explicit rules. The audit methodology focused on evaluating existing rules for over-permission rather than validating every required access path end-to-end. SSH to Heimdall was not an active workflow during the audit, so it was never tested and the gap went undetected.

# **What We Could Have Done Better**

## **Access Matrix Validation**

Every firewall hardening session should conclude with a structured test of every required access path — not just a visual review of rules. If it is not tested, it should not be assumed to work. Going forward, any hardening session will include an end-to-end validation step before closing.

## **DNS Validation at VLAN Build Time**

When the SOC VLAN was stood up, DNS resolution from soc-stack should have been verified against the intended resolver (10.0.10.1) before moving on. The issue was present from day one but was masked by the fallback behavior of querying 192.168.100.1 directly, which generated block logs rather than a hard failure.

## **Known Pending Items Left Open Too Long**

The soc-stack DNS issue was identified and documented on March 28 but not resolved until March 30. The stale Heimdall route (10.99.0.0/24 via 192.168.100.2) has been open since early March. Pending items should be prioritized and closed in order rather than carried forward indefinitely.

## **Post-Hardening Verification Checklist**

The firewall audit in February did not include a test phase. Future hardening sessions require a verification pass that explicitly tests every device-to-device path that is expected to work. This should be documented as a checklist and signed off before the session is closed.

# **Process Improvements**

- **Implement end-to-end access matrix validation after every firewall change session**

- Test every required access path — do not assume rules that look correct actually work

- Include SSH, DNS, web UI, and syslog forwarding in the test matrix for each VLAN

- **Validate DNS resolution from every new VM or VLAN at build time**

- Run dig @<gateway> google.com and dig @<gateway> <internal domain> before closing the session

- **Maintain a living pending items list and work it in priority order**

- Do not close sessions with known open items unless they are explicitly deferred with a reason

- **Document out-of-band access paths before they are needed**

- Tailscale on Heimdall was the recovery path today — this should be documented proactively, not discovered reactively

# **Remaining Open Items**

| **Item** | **Notes** | **Status** |
| --- | --- | --- |
| Fix stale Heimdall route — 10.99.0.0/24 via 192.168.100.2 → via 192.168.100.250 | NM connection file on Heimdall, route2 field | **Open** |
| Deploy Wazuh Manager-only on pve-SOC | 4GB VM, feeds existing Elasticsearch | **Open** |
| Disable root password SSH on Proxmox nodes (key-only) | Part of general hardening session | **Open** |
| Suricata tuning and IPS mode | After hardening session | **Open** |
| soc-stack DNS fix | Resolved this session | **Closed** |
| SSH to Heimdall from LAN (firewall gap) | Discovered and resolved this session | **Closed** |

github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution
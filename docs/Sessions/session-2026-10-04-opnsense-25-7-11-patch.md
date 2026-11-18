---
name: session-2026-10-04-opnsense-25-7-11-patch
type: session
date: 2026-10-04
---

Raw log of the OPNsense patch update and the checks run afterward. Started as a question about whether to build the Kea DHCP server in the same change as the `.local` to `.internal` migration. See [[dhcp-dns-rollout]] and [[dns-architecture]].

## Decisions
- Stay on `.internal`. Zone apex `homelab.internal` agreed by Daniel. Daniel briefly considered `homelab.lan`, then chose `.internal` (`.lan` is not reserved for private use). Recorded in [[dns-architecture]].
- Finish the OPNsense upgrades before building Kea. Kea configuration waits on [[opnsense-major-upgrade]].
- Kea needs a pool on every subnet per the OPNsense Kea documentation, so reservation-only VLANs get a small pool. The documentation page was the current one, not 25.7 specific, so confirm in the UI.

## What was done
- Configuration backup downloaded before the update (System, Configuration, Backups).
- [[opnsense]] updated from 25.7.10 to 25.7.11_9. Packages: opnsense, opnsense-update, libsodium 1.0.21, suricata 8.0.3, new package hostwatch 1.0.6 (the host discovery service; the 26.1 release notes say it is enabled by default and can be turned off under Interfaces, Neighbors, Automatic Discovery).
- Box rebooted and came back up. `opnsense-version` on the admin session reports 25.7.11_9 (amd64).
- The install is on UFS. System, Snapshots does not exist, and the OPNsense documentation says snapshots require ZFS. No snapshot rollback is available.

## Checks after the update
- WAN up, interface IPs intact, Intrusion Detection still shown as enabled, firewall rule order looks unchanged (all by eye, per Daniel).
- DNS tests passed except `.local` runs from [[ubuntu-401]] (see below).
- Suricata events reaching Kibana after the reboot: confirmed at about 10:58. Data view `suricata-logs-*`, last 15 minutes showed 13 alert events, all in a burst around 10:47 on interface em0, none after. The last 24 hours (108 events) show sparse, bursty alerting with long empty stretches both before and after the update, and no unexplained gap around the reboot. Read from a screenshot with 30 minute buckets, so times are approximate. Several buckets hold exactly 13 events, which looks like one repeating source (possible tuning item for the SIEM work).
- The syslog lines carry the hostname `OPNsense.homelab.local`, so OPNsense's own system domain is still `homelab.local`. Add it to the Phase 1 rename list in [[dhcp-dns-rollout]].

## 25.7 is end of life
- Per endoflife.date, 25.7 reached end of life on 2026-01-28 and 26.1 on 2026-07-15. Only the latest release is supported. 26.7 is current (26.7.4_1 released 2026-09-15 per the OPNsense release notes).
- The update hold on 25.7.11 (Unbound DNSBL regression) ran roughly nine months. [[opnsense-check-apply-updates]] has no limit on how long a hold can last.
- Plan: 25.7 to 26.1, then likely 26.7. Whether 25.7 can go straight to 26.7 is not verified. See [[opnsense-major-upgrade]].

## DNS findings (from ubuntu-401, VLAN30)
- `dig env1.homelab.local` through the local resolver stub (127.0.0.53) returned SERVFAIL in 0 ms with the aa flag. The query never left the machine. Consistent with the stub handling `.local` itself. The mechanism was not verified against systemd documentation.
- `dig @10.0.30.x` (Unbound) returned NXDOMAIN with the root servers' SOA. Consistent with Unbound recursing for `homelab.local` and having no forwarding rule. The changelog records that forwarding was removed to break a Pi-hole and Unbound loop, so this predates the update.
- `dig @192.168.100.x` (Pi-hole) returned 10.0.20.x for `env1.homelab.local`. That is a Pi-hole record pointing at the internal NPM on services-host, intentional per Daniel.
- Result: Pi-hole is the only place `.local` names resolve today. Clients that use Unbound (VLAN30 and VLAN51 per the vault) cannot resolve them. Phase 1 of [[dhcp-dns-rollout]] moves internal names into Unbound.
- Each migrated name behind NPM also needs an NPM proxy host and a certificate for the new `.internal` name.

## Agentless monitoring of OPNsense is not working
- `/var/ossec/agentless/.passlist` on [[wazuh-manager]] holds two lines: `root@10.0.0.x NOPASS` and a base64 line that decodes to `root@10.0.0.x|NOPASS|`.
- A manual SSH as root to 10.0.0.x with the Wazuh key is refused (publickey). Root from the workstation is refused too. Login as `admin` works.
- `sudo sshd -T` on OPNsense: `permitrootlogin no`, `pubkeyauthentication yes`, `passwordauthentication no`.
- Conclusion: the agentless check logs in as root, and root SSH is off, so it cannot authenticate. The current `ossec.log` has no agentless or ssh_integrity lines. When it stopped working is not known. Rotated logs were not searched at the time of writing.
- The runbook said the passlist file is on OPNsense. It is on the Wazuh manager. Corrected in [[opnsense-agentless-ssh-monitoring-setup]].
- Daniel chose the `os-wazuh-agent` plugin direction at 11:02. Not installed yet, and Daniel decided to install it after the upgrades. `pkg search wazuh-agent` on OPNsense 25.7.11_9 shows os-wazuh-agent 1.2_3, os-wazuh-agent-devel 1.3 and wazuh-agent 4.12.0. The manager is 4.14.4, so the agent would be older than the manager. The FreeBSD port is at 4.14.6 (quarterly) and 4.14.8 (latest) per freshports, so a later OPNsense release may offer an agent newer than the manager. Check `pkg info wazuh-agent` after each upgrade hop.
- Options discussed: a dedicated unprivileged check account (unverified that the check works without root), the official `os-wazuh-agent` plugin (limited community support, version compatibility with the 4.14.4 manager not checked), or re-enabling root with key only (undoes hardening, not recommended).

## Open
- Agentless monitoring gap: choose a fix.
- Search rotated Wazuh logs for the last successful agentless run.
- Decide whether the update hold policy needs a limit.
- Firewall hygiene noticed in rule screenshots (Oct 4, 12:01, not changed): the DEV interface has "Allow Services to Authentik" with source Services net. Traffic from Services net does not arrive on the DEV interface, so it looks like a dead duplicate of the Services rule. Confirm before removing. The SOC interface still has "Allow Wazuh agentless SSH to OPNSense" (Wazuh_Manager to This Firewall, port 22) although the agentless check cannot authenticate. Remove it when the Wazuh agent plugin replaces agentless monitoring.

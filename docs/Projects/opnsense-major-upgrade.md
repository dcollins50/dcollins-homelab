---
name: opnsense-major-upgrade
type: project
status: in-progress
---

Move [[opnsense]] off the end-of-life 25.7 series. Planned Oct 4, 2026. Hop 1 (25.7.11_9 to 26.1) applied Oct 4, 2026, `opnsense-version` reported 26.1.2. Same day, patched within the 26.1 series to 26.1.11_10, fully verified. Hop 2 (26.7) not started, Daniel's plan is to soak a few days before it. Anything marked unverified has not been checked.

## Why
- Per endoflife.date, 25.7 reached end of life on 2026-01-28 and 26.1 on 2026-07-15. Only the latest release is supported.
- The box is on 25.7.11_9 as of Oct 4, 2026 ([[session-2026-10-04-opnsense-25-7-11-patch]]).
- Kea DHCP work in [[dhcp-dns-rollout]] waits on this, so Kea is built once on the release it will stay on.

## Constraints
- UFS install. No System, Snapshots. Rollback means a fresh install plus the configuration XML. Unverified: whether a 25.7 configuration restores cleanly into a newer release.
- Unverified: whether 25.7 can go directly to 26.7. Plan on stepping through 26.1.
- All routed traffic drops during each reboot. Remote paths behind OPNsense go down with it. Do this at the box with a keyboard and monitor available, workstation on its direct switch port.

## Known risks per release (from the OPNsense release notes)
**26.1**
- The rules page moves to a new interface with a migration tool. The notes warn that moving a rule off the floating interface changes its priority. Do not run the migration blindly.
- Suricata 8: the old `custom.yaml` is removed in favor of a `conf.d` directory. Only matters if one was ever added.
- Port Forwarding is renamed Destination NAT. No inbound port forwards here.
- ISC DHCP moves to a plugin and the default becomes Dnsmasq. No DHCP runs on OPNsense today.
- `mwexec()` and `mwexec_bg()` are removed. Only matters for custom scripts.

**26.7**
- Outbound NAT becomes Source NAT, with a migration assistant.
- Remove third-party repositories and the CPU microcode plugin first. OpenSSL moves from 3.0 to 3.5.
- Firewall rule pages move to the os-firewall-legacy plugin. Rules keep working without it.
- Adds Kea DDNS, custom options and dynamic prefix delegation.

## Prep status (Oct 4, 2026)
- Suricata `custom.yaml`: checked on the box. It is an empty stub ("empty stub for custom modifications, add custom persistent config below"), 88 bytes, nothing of Daniel's in it. `conf.d` exists and holds only a README. Nothing to migrate for the 26.1 change.
- Nothing is configured outside the GUI (Daniel, Oct 4).
- The config backup XML is password-protected, so it cannot be parsed for a rule baseline. Baseline instead from the loaded ruleset on the box (`pfctl -sr`), planned, not yet run.
- Daniel wants the 26.1 hop as soon as possible. Only hop 1 first, then a soak of a few days before 26.7.
- The os-wazuh-agent plugin is installed after the upgrades, not before (Daniel, Oct 4). Check the repo's wazuh-agent version against the manager then.
- USB installer: the OPNsense download page offers the current release (26.7). It says nothing about older versions, and the mirror listing could not be read, so a 25.7 image is not confirmed. Plan on the 26.7 VGA image. The site warns that checksum files next to images do not prove authenticity, so verify as the download page describes. Restoring a 25.7 config into 26.7 is unverified.
- The 26.1 notes say the firewall migration page is not something to jump into right away. Leave the rules alone during the hop.
- Firewall baseline captured Oct 4, 2026: `pfctl -sr` saved as `~/pf-rules-before-26.1.txt` on the box (163 lines) and copied to Daniel's Downloads folder. A repeat run before any change produced an empty diff, so the method is stable.
- System, Firmware, Updates (Oct 4, 11:37) offers a direct upgrade from 25.7.11 to 26.1 (base, kernel, packages), 1.2 GiB, reboot required. Not applied yet.
- USB installer: 26.7 VGA image written to a USB stick (Daniel, Oct 4, 11:47). Boot not tested, checksum verification not confirmed. Daniel also has the 25.7 DVD ISO downloaded Jan 5, 2026, not confirmed to boot from USB.
- Plugins and repositories checked on the box (Oct 4, 11:49): `pkg info | grep -i -E "^os-|microcode"` returned nothing, so no plugins and no microcode package installed. Repo files are FreeBSD.conf (FreeBSD and FreeBSD-kmods both `enabled: no`) and OPNsense.conf (`enabled: yes`), no third-party repos.
- Open: a fresh config backup right before the upgrade, the exact window. Daniel can reach the box physically if needed.

## Before each hop
- Fresh configuration backup, copied off the box.
- Screenshot or export firewall rule order per interface.
- USB installer for the current release on hand.
- Read the release notes for that release in full.
- Check for anything configured outside the GUI.

## After each hop
- WAN up, VLAN interfaces up with correct IPs.
- Firewall rule order matches the screenshots.
- Unbound answers on each VLAN, external and internal names. See [[session-2026-10-04-opnsense-25-7-11-patch]] for the expected current results.
- Suricata running and events reaching Kibana.
- Agentless monitoring status, once the gap in that session note is resolved.
- Leave a few days between hops.

## Related
- [[opnsense-check-apply-updates]]
- [[dhcp-dns-rollout]]
- [[dns-architecture]]

## Hop 1 result (Oct 4, 2026, about 12:06)
- `opnsense-version`: OPNsense 26.1.2 (amd64). The 26.1 series was at 26.1.11 per the release notes, so patches within 26.1 were still outstanding at this point.
- Loaded ruleset after the hop vs the baseline: 163 lines, two differ, an auto-generated table name and a regenerated label on the IPv6 block rule. Rule content unchanged.
- Existing rules now show as legacy rules and keep being enforced. A later, deliberate migration is a separate task. The migration assistant was not used.

## Hop 1 post-upgrade checks, confirmed (Oct 4, 2026, afternoon)
- WAN (em0) up at 192.168.100.x. All nine VLAN interfaces (vlan01 to vlan09) up with the documented IPs: 10.99.0.x (VLAN40), 10.98.0.x (VLAN41), 10.0.0.x (mgmt), 10.0.50.x (DMZ1), 10.0.20.x (Services), 10.0.30.x (DEV), 10.0.60.x (Storage), 10.0.10.x (SOC), 10.0.51.x (DMZBastion).
- Unbound answers correctly from VLAN30: external names resolve, and `authentik.homelab.example` resolves to its internal override (10.0.20.x).
- Suricata events reaching Kibana: confirmed via an `ET POLICY GNU...` alert in `suricata-logs-*` at 11:31, sourced from 192.168.100.x (OPNsense's WAN address) to an external IP on port 80. Full chain (Suricata, EVE JSON, Logstash, Elasticsearch) working as of that alert.
- No DHCP or RA service actually serving: `dhcpd` (legacy ISC) and `dnsmasq` processes are both running but idle, no DHCP ranges configured on either, matches the documented state (no DHCP built yet, Kea DHCP is a future project in [[dhcp-dns-rollout]]).

## 26.1 point-release patch (Oct 4, 2026, evening)
- Check for updates offered 26.1.2 to 26.1.11_10 (base, kernel, and a large package set). Notable items: Suricata 8.0.3_1 to 8.0.6, Unbound 1.24.2_1 to 1.25.1_1, dnsmasq 2.92,1 to 2.93,1, os-isc-dhcp 1.0_3 to 1.0_6, OpenSSL and OpenSSH point bumps, OpenVPN 2.6.19 to 2.7.5 (no active OpenVPN config on this box). A large set of new `py313-*` packages came from a base Python 3.11 to 3.13 upgrade at 26.1.3 (March 2026), not a new feature, confirmed via the OPNsense 26.1 release notes.
- 26.1.8 patched a real CERT vulnerability in dnsmasq (VU#598349), a genuine security reason to apply, not just version hygiene.
- 26.1.5 through 26.1.8 added real functionality to Kea DHCP (dynamic prefix delegation, DDNS, reservation improvements), relevant later for [[dhcp-dns-rollout]].
- Applied. `opnsense-version` confirms OPNsense 26.1.11_10 (amd64).
- Post-patch ruleset diff against the original pre-hop-1 baseline (`pf-rules-before-26.1.txt`): same two noise lines as after hop 1 (auto-generated table name, regenerated IPv6 block rule label), nothing else changed.
- Re-checked after the patch: DHCP/RA processes still idle, no ranges configured. Unbound still resolves `authentik.homelab.example` to 10.0.20.x correctly.
- Hop 1 is now fully verified and patched to the latest 26.1 point release. Plan: soak a few days before starting hop 2 (26.7).

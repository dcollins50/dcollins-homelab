---
name: incident-2026-10-21-php2shell-php-rce
type: incident
date: 2026-10-21
status: open
---

Two critical remote code execution flaws in the PHP interpreter itself ("PHP2Shell", CVE-2026-109618 and CVE-2026-109619, CVSS v4.0 9.2), fixed in an emergency PHP release on Oct 21, 2026 (8.5.12, 8.4.27, 8.3.36, 8.2.35). Exploitation in the wild was reported the same day. PHP's own developers say patching is the only real fix; ini and config changes only reduce specific attack paths. Apache mod_php is the highest-risk setup, and the exploit can run before application code, so a login does not protect the app.

## Exposure check
| Host | Uses PHP | Result |
|---|---|---|
| [[nimbus]] | Yes, Nextcloud 34 on Apache with PHP 8.4 (Debian 13) | Patched Oct 21, 2026 (Debian fix `8.4.27-1~deb13u1`, DSA-6574-1). Daniel confirmed. Tailnet only, so exposure was limited to tailnet members. |
| [[opnsense]] | Yes, GUI is PHP-based | **Open.** `php -v` showed 8.3.31 (built Jun 29, 2026). Fixed 8.3 release is 8.3.36. Waiting on OPNsense to ship a package with the fix. No OPNsense statement found as of Oct 21. GUI is LAN only, no WAN port forwards. |
| [[heimdall]] | No | Pi-hole v6 (core 6.5, web 6.6.1, FTL 6.8) has no PHP. Updated Oct 21 anyway. |
| [[dvwa]], [[metasploitable2]] | Yes, old PHP | Intentionally vulnerable and isolated on [[VLAN40]]. No fix exists or is wanted. |
| Everything else | No | Gitea, Vaultwarden, Authentik, ntfy and the rest do not use PHP. |

## Still to do
- Check System, Firmware, Updates on [[opnsense]] daily until a revision with PHP 8.3.36 appears, then update and re-check `php -v`.
- Narrow who can reach the OPNsense GUI (System, Settings, Administration listen interfaces, plus LAN and VLAN rules allowing 443 to 10.0.0.x). The Sep nmap from Kali found 443 open, so confirm that did not come from a lab VLAN.
- The update-hold policy in [[opnsense]] does not help here, because the flaw is in the version already running, not in a new release.
- Do not run public PoC code for this CVE. Tenable warns unverified exploit code circulating now can contain malware.

## Notes
- Suricata on [[opnsense]] is IDS only and on the WAN only, so it would not see LAN-side attempts. Agentless Wazuh monitoring of OPNsense was already broken (see the Oct 4 gaps in [[Conventions]]).
- Confirm on [[nimbus]] that Apache was restarted after the upgrade, since mod_php keeps the old code loaded until then.

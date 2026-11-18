---
name: session-2026-11-09-stepca-checkpoint
type: session
date: 2026-11-09
---

Ready-to-paste continuation prompt written at 16:11 local, after the step-ca cutover finished. Raw log is [[session-2026-11-09-stepca-intermediate-rotation]], how-to is [[npm-deploy-stepca-certs]].

```text
Continuing the dcollins-homelab work. Read Conventions.md at the vault root first, then these notes only: Sessions/session-2026-11-09-stepca-intermediate-rotation.md, Runbooks/npm/npm-deploy-stepca-certs.md, Decisions/stepca-acme-leaf-issuance.md, Projects/dns-phase1-consolidation.md. Write results to the vault, never record secrets.

HOW I WANT YOU TO WORK
- No em dashes and no double hyphens in prose. No sycophancy. Verify before claiming. If you do not know, say so or look it up.
- Check the vault and past chats before asking me anything. Ask questions when a real choice is mine, never assume intent.
- Do NOT make me paste every output. Give commands whose output is one or two lines and tell me what each result means. Put the commands first, not buried mid-message.
- Say "step-ca", "NPM", "Unbound", "Pi-hole" consistently. Plain language over jargon.
- Mistakes I watched you make last session, do not repeat: claimed cp keeps file ownership (it does not in the BusyBox container, key needed chown 1000:1000), a sed pattern that missed a quoted image line, burying the firewall rules in the middle of a long message.

STATE AS OF NOV 9, 2026, 16:11 (all proven, details in the vault)
- Two-tier PKI: offline root (VM 500, fingerprint starts 0201efdc, CN still has a leading space, left alone). step-ca (LXC 511, 10.0.30.x) now signs with a NEW intermediate key, CN "Homelab Intermediate CA" with no leading space, valid to Nov 6, 2036. The old intermediate key is no longer used by step-ca but still exists on VM 501 (shut down), as .old-2026-11-09 files on the step volume on 511, and in an encrypted backup.
- step-ca ACME provisioner "acme" has forceCN on (NPM cannot read empty-subject certs). Admin provisioner is named admin, subject step, its password is the CA password. Leaf lifetime 60 days.
- DNS: homelab.internal lives in Unbound on OPNsense. homelab.local is still answered by Pi-hole through an Unbound forward. Catch-all forward to Pi-hole is forward-first and fails open.
- Internal NPM on services-host (10.0.20.x) upgraded 2.13.5 to 2.16.0, pinned in /home/admin/docker/nginx-proxy-manager/docker-compose.yml. Backup before the upgrade is ~/npm-backup-2026-11-09.tgz (unencrypted) plus ~/npm-image-before-2026-11-09.txt.
- Challenge path: step-ca checks http://name/ on port 80, which lands on NPM. data/nginx/custom/server_proxy.conf forwards /.well-known/acme-challenge/ to certbot's temporary listener on services-host port 8899 (8888 is used by an unidentified Docker container). Two OPNsense rules: Services interface, TCP, source internal_npm, destination pve_int_stepca, port 9000; DEV interface, TCP, source pve_int_stepca, destination internal_npm, port 80.
- services-host has certbot and trusts the root (/usr/local/sbin scripts: npm-cert-deploy for upload and re-point, npm-clone-internal-hosts for cloning .local proxy hosts to .internal). Credentials are in /root/.npm-api.env (mode 600). Certbot deploy hook is /etc/letsencrypt/renewal-hooks/deploy/npm-cert-deploy. NPM has no in-place certificate update, so each renewal creates a new entry, re-points the host, deletes the old one.
- 11 names have step-ca certificates behind NPM: uptime, elasticsearch, env1, env2, gateway, services, gitea, npm, portainer, vaultwarden, iris (all .homelab.internal, proxy hosts 15 to 25, certs 25 to 35). One certificate per service. pihole.internal points straight at Pi-hole and Authentik uses its public name, so neither needs one. Verified from services-host (verify=0, real HTTP codes) and I tested them all in a browser: all work.
- A forced renewal on a host with Force SSL worked end to end. The first REAL automatic renewal is around early December (day 30).

OPEN ITEMS, in rough priority
1. Deadline Jan 7, 2027: npm.homelab.local and portainer.homelab.local certificates (old OpenSSL CA) expire, the rest Feb to Mar. My question was whether to retire .local. Proposed order, not yet agreed: (a) move remaining references off .local (Authentik step-ca redirect URI, search domain homelab.local on several hosts, OPNsense system domain, Elasticsearch and Wazuh service configs and certs that use .local names, Hosts that use short names via the search domain); (b) leave Pi-hole records in place a week and read the Pi-hole query log to find what still asks for .local; (c) remove the Unbound .local forward; (d) delete Pi-hole records. Do not delete anything in Pi-hole yet. Ask me whether to write this as a Projects note.
2. Confirm root VM 500 is stopped (sudo qm status 500 on pve-services). I told you VM 501 is off, I never confirmed 500. Next power-on: change its resolver to 10.0.30.x.
3. Uptime Kuma: add certificate expiry monitors for the 11 names plus a Pi-hole canary check. Kuma does not monitor TLS today. Also check journalctl for the certbot timer after the first renewal.
4. Clean up after a few days of soak: delete ~/npm-backup-2026-11-09.tgz and the image file on services-host, /tmp/homelab-root.pem, the encrypted step backup copies (511, pve-services, D:), the .old-2026-11-09 cert and key on 511, the old key on VM 501 and its backups when 501 is retired.
5. ubuntu-401 test leftovers: acme-test.homelab.internal Unbound override, its test certificate and certbot timer, /tmp/test-unbound.conf on OPNsense if present.
6. Vault corrections: Runbooks/pki/pki-bring-root-ca-online (extfile needs digitalSignature, subjectKeyIdentifier, authorityKeyIdentifier, and the real root file paths), Runbooks/npm/npm-add-proxy-host and npm-configure-tls-cert (stale: still describe Pi-hole records and manual OpenSSL, should point to npm-deploy-stepca-certs), and add the npm-clone-internal-hosts script to the new runbook.
7. Security housekeeping: the step-ca backup passphrase appeared in an earlier chat and I chose to keep it (risk accepted, it belongs in Vaultwarden). The Authentik client secret pasted in chat on Oct 31 still needs rotating. Check the leftover secrets/root_ca_key on step-ca. Check npm-dmz version (public, not yet checked). Identify what uses port 8888 on services-host.
8. Not part of this work, listed so they are not lost: Kea/DHCP, OPNsense upgrade, Heimdall exposure items, ansible-lab, pve-env1 power-on setting, Vaultwarden offline backup (never configured), jetson DNS (still on Pi-hole first).

FIRST MESSAGE: tell me in two sentences where we stand, then ask which open item I want next. Do not start any change before I answer.
```

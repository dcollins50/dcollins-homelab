---
name: session-2026-11-09-stepca-intermediate-rotation
type: session
date: 2026-11-09
---

Raw log. Continues [[stepca-acme-leaf-issuance]] and [[sop-stepca-cutover]]. Phase 1 DNS was already done on Oct 31 ([[dns-phase1-consolidation]]), so the plan from Oct 29 was not rerun.

## Decisions (Daniel)
- Stay on NPM and use a workaround for renewals into NPM.
- Include a new intermediate key for [[pve-int-stepca]] with the leading space in the CN fixed.
- Order: backups, new intermediate, `forceCN` on the `acme` provisioner, NPM workaround. Whether to upgrade NPM (2.13.5) before building the workaround is still open.

## Facts read today
- Current intermediate (read from [[pve-ca-intermediate]]): RSA 4096, sha256WithRSA, valid Dec 27, 2025 to Dec 25, 2035, `CA:TRUE, pathlen:0`, key usage Digital Signature, Certificate Sign, CRL Sign. Subject has no `L=Mansfield`.
- Root ([[pve-ca-root]]): valid Dec 27, 2025 to Dec 22, 2045, fingerprint SHA256 `02:01:EF:DC...7E:45`, matching the one step-ca logged. The root CN also has a leading space (`" Homelab Root CA"`). Fixing that means replacing the root, so it stays.
- [[pki-bring-root-ca-online]] gives key usage `keyCertSign,cRLSign` for signing the intermediate. The certificate in service also has Digital Signature. Runbook not corrected yet.
- step-ca data is the Docker volume `step` at `/home/step`. `secrets/root_ca_key` is 314 bytes, so it is not the real RSA 4096 root key (size inference only).
- The SSH CA keys are probably encrypted with the same `secrets/password` file as the intermediate key (not verified). Plan: do not change that file, encrypt the new intermediate key with the same passphrase.

## Backup of step-ca (10:00 local)
- On LXC 511, as root: `docker stop step-ca`, tar of the volume to `/root/step-backup-2026-11-09.tgz` (84161 bytes), `docker start step-ca`. step-ca was down about a minute. Status right after the start was `Up Less than a second (health: starting)`; healthy status not yet confirmed.
- Encrypted with `openssl enc -aes-256-cbc -pbkdf2` and listed: contains `config/ca.json` plus its two backups, all five files in `secrets/`, and `db/MANIFEST`.
- Backup passphrase value is not recorded in the vault. It belongs in Vaultwarden.

## What went wrong
- The generated backup passphrase was included in Daniel's paste into the chat, and was repeated back at his request. Daniel chose to keep it (10:05), so the archive was not re-encrypted. Anyone with this chat history plus the encrypted file could open it; the file is on the workstation D: drive (BitLocker).

## Backup copied (10:13)
Encrypted archive copied to `D:\step-backup-2026-11-09.tgz.enc`. SHA256 `90b475bb2704e42d05ea1c133cee05a314ebacbd669b4ef4d5b85d7c4c6d528f` is identical on pve-int-stepca, pve-services and the workstation. Cleanup of the plaintext `.tgz` on LXC 511 and the copy in the admin home on pve-services was started after this. The encrypted copy stays on 511 until the rotation is finished, since it adds no exposure beyond the volume on the same box.

## New intermediate signed (10:27 local)
- CSR generated on LXC 511 with `openssl req -new -newkey rsa:4096`, key encrypted with the existing step-ca passphrase (read from the `password` file, not typed). CSR public key hash `02b837108f8497ac6d3baf24c746085904470708a6b0226191f901842ffee49e`. Subject `C=US, ST=Texas, O=Homelab, OU=Certificate Authority, CN=Homelab Intermediate CA, emailAddress=admin@homelab.local`.
- The old intermediate's extensions were read first: subject key ID, authority key ID `C6:40:0A:38...`, `CA:TRUE, pathlen:0` (critical), key usage Digital Signature, Certificate Sign, CRL Sign (critical), no name constraints, no CRL distribution points.
- Root VM 500 started with `qm start 500` from pve-services. Clock correct (16:24:59 UTC), root fingerprint matched in full. Files: `~/homelab-ca/root-ca/private/root-ca.key` and `~/homelab-ca/root-ca/certs/root-ca.crt`. No `openssl*.cnf` found under the home directory to depth 4.
- Signed at 16:27 UTC with `openssl x509 -req ... -days 3650 -sha256` and an extfile mirroring those extensions (`authorityKeyIdentifier=keyid`). Result: `verify` OK, valid Nov 9, 2026 to Nov 6, 2036, same authority key ID as the old intermediate, subject without the leading space, issuer still the root's `\ Homelab Root CA`, public key hash equal to the CSR's.
- Copied back to pve-services and pushed to `/root/new_intermediate_ca.crt` on LXC 511. Public key hash equal at both ends (10:30).
- Mistake: the first `scp` was run on the root VM, where the CSR did not exist, because the command block put `scp` and `ssh` together. No impact.
- Root VM resolver read: Pi-hole `192.168.100.x` then `1.1.1.1`, search `homelab.local`. Not changed, see [[pve-ca-root]].

## Installed into step-ca (16:37 GMT, 10:37 local)
- Stopped step-ca, kept `certs/intermediate_ca.crt.old-2026-11-09` and `secrets/intermediate_ca_key.old-2026-11-09` on the volume, copied the new cert and key over the originals, started step-ca.
- Mistake: I said `cp` onto an existing file keeps owner and permissions. BusyBox `cp` replaced the files, so the new key was `root:1000` mode 600 and step-ca (uid 1000) looped on `permission denied`. Fix: `chown 1000:1000 /home/step/secrets/intermediate_ca_key` then `docker restart step-ca`. The import step in [[pki-bring-root-ca-online]] style runbooks already had the chown; I left it out. Downtime was a few minutes.
- After the fix: `Serving HTTPS on :9000`, status `(healthy)`, root fingerprint `0201efdc...77e45` unchanged.

## Proof by ACME (16:39 GMT)
- `certbot renew --force-renewal` on [[ubuntu-401]] for `acme-test.homelab.internal` succeeded. Leaf issuer `CN=Homelab Intermediate CA` (no leading space), notBefore Nov 9 16:39:06 2026, notAfter Jan 8 2027 (60 days), empty subject (no `forceCN` yet).
- Chain: leaf, then the new intermediate, issued by `" Homelab Root CA"`. `openssl verify` against the root in the system trust store: OK.

## forceCN enabled (10:50 local)
- Ran `docker exec -it step-ca step ca provisioner update acme --force-cn --admin-subject step --admin-provisioner admin` on pve-int-stepca. The admin provisioner is named `admin` (JWK), subject `step`, and its password is the step-ca CA password.
- Proven: `provisioner list` shows `"forceCN": true` once, and a forced certbot renewal on [[ubuntu-401]] now gives `subject=CN = acme-test.homelab.internal` instead of an empty subject.
- Root VM 500 shutdown was never confirmed in output. Daniel reported "vm 501 is shut down" (the old intermediate). Check 500 at the next pve-services visit.

## NPM upgraded 2.13.5 to 2.16.0 (11:06 local)
- Why: the Oct 31 research recorded CVE-2026-40519 (authenticated RCE) as affecting 2.13.5. A third-party write-up says 2.16.0 fixes it and that 2.15.1 still had the bug. The API the renewal script will use may also differ from 2.13.5, so the script is built after the upgrade.
- Backup first, with NPM stopped: `~/npm-backup-2026-11-09.tgz` (data and letsencrypt, 1483968 bytes, unencrypted) and `~/npm-image-before-2026-11-09.txt` (the old image ID) in the admin home on [[services-host]]. Delete the archive after a few clean days.
- Pulled `latest`, which resolved to 2.16.0. `curl http://127.0.0.1:81/api/` returned version 2.16.0. Jumped straight from 2.13.5; the write-up only tested 2.15.1 to 2.16.0.
- Mistake: my `sed` to pin the tag did not match because the compose image line is quoted (`'jc21/nginx-proxy-manager:latest'`). The upgrade still worked because `latest` was 2.16.0. Pinning to `2.16.0` was given as a second command.
- Rollback: stop, restore the backup archive, start the old image. Do not just change the tag back (a downgrade followed by saving a host can drop per-path access rules).
- 11:09: tag pinned to `2.16.0` in the compose file and the container recreated. Daniel confirmed the proxied sites still load. Old API tokens are invalid after the upgrade.

## NPM certificate path proven on `uptime.homelab.internal` (13:55 local)
- Decisions (Daniel): one certificate per service, new separate `.internal` proxy hosts alongside the `.local` ones. Eleven names point at NPM (elasticsearch, env1, env2, gateway, services, gitea, npm, portainer, uptime, vaultwarden, iris); pihole.internal points straight at Pi-hole and Authentik uses its public name, so no certificate for those two.
- Built: `server_proxy.conf` challenge forward, port 8899, two OPNsense rules, certbot plus root trust on [[services-host]], the `npm-cert-deploy` script and certbot deploy hook. Full details and the script are in [[npm-deploy-stepca-certs]].
- Proven: certbot issued the cert through NPM's port 80, the script uploaded it and attached it to proxy host 15, `curl` returned `302 verify=0`, and a forced renewal with Force SSL on replaced cert 24 with 25.
- Mistakes: (1) I buried the two OPNsense rules mid-message and Daniel ran the later commands without them. (2) Rule 1 was entered with source and destination swapped (visible in his screenshot), about 40 minutes lost; OPNsense logs showed nothing even with logging on, because the traffic matched no rule. (3) The first `curl` check after the script ran before NPM reloaded and printed `000 verify=1`; I first blamed a missing intermediate, which the next command disproved.
- Port 8888 on services-host is used by a Docker container (not identified), so the helper uses 8899.

## Ten more `.internal` hosts and certificates (14:00 local)
- `npm-clone-internal-hosts` (copies each `.local` proxy host to its `.internal` name with no certificate, `/usr/local/sbin/`, same credentials file) created proxy hosts 16 to 25. Then certbot plus `npm-cert-deploy` issued and attached certificates 26 to 35 for elasticsearch, env1, env2, gateway, services, gitea, npm, portainer, vaultwarden and iris. All 10 succeeded on the first try. With uptime that is 11 names, each a separate 60-day certificate. Clone script is not yet saved in the runbook.
- Verified 14:03 from services-host: all 11 `.internal` names return a real HTTP code with `verify=0` (uptime 302, elasticsearch 302, iris 302, the other eight 200). Browser check 14:04: `https://vaultwarden.homelab.internal` loads from the workstation with no certificate warning. Daniel tested all the `.internal` names at 14:07 and reports they all work. Not checked from another VLAN. The `.local` hosts still use the old OpenSSL certificates (npm and portainer expire Jan 7, 2027).

## Next
- Confirm root VM 500 is stopped (`sudo qm status 500`).
- Create the other 10 `.internal` proxy hosts, issue and deploy their certificates (deadline Jan 7, 2027 for npm and portainer `.local`, see [[stepca-acme-leaf-issuance]]).
- Retire the old intermediate key after a soak: `.old-2026-11-09` files on 511, the copy on [[pve-ca-intermediate]], and its backups.
- Fix [[pki-bring-root-ca-online]]: extfile needs digitalSignature, subjectKeyIdentifier, authorityKeyIdentifier, and the real root file paths.

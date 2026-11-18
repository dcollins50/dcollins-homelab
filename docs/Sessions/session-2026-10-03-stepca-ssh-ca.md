---
name: session-2026-10-03-stepca-ssh-ca
type: session
date: 2026-10-03
---

Continued from [[session-2026-10-03-bastion-warp-ipv6]]. Goal: let [[pve-bastion]] sshd trust step-ca SSH user certificates. Host: [[pve-int-stepca]].

## Finding: step-ca had no SSH CA configured
- `docker exec step-ca step ssh config --roots` returned "step certificates is not configured with SSH support", even though the Authentik OIDC provisioner has `enableSSHCA` true.
- Inside the container, `/home/step/secrets` held only `intermediate_ca_key`, `root_ca_key` and `password`, and `ca.json` had no `ssh` block. The Sep 21 rebuild from the imported intermediate never created SSH keys.
- Smallstep's docs only show `step ca init --ssh` for new CAs. The retrofit steps below come from a GitHub discussion (smallstep/certificates #400) and a third party blog post, not from Smallstep's docs.

## What was done
1. Backed up `/home/step/config/ca.json` as `ca.json.bak-2026-10-03` (same directory).
2. Generated two key pairs inside the container with the CA's existing password file, EC P-256 by default: `step crypto keypair /home/step/certs/ssh_host_ca_key.pub /home/step/secrets/ssh_host_ca_key --password-file /home/step/secrets/password`, and the same with `ssh_user_ca_key`. No `--force` used.
3. Inserted one line after `"federatedRoots": null,` in `ca.json`: `"ssh": {"hostKey": "/home/step/secrets/ssh_host_ca_key", "userKey": "/home/step/secrets/ssh_user_ca_key"},` using `sed -i` with a `[[:space:]]` pattern (the image is busybox).
4. `docker restart step-ca`. Container came back healthy.
5. `docker exec step-ca step ssh config --roots` now prints the SSH user CA public key:

```
ecdsa-sha2-nistp256 [public-key]
```

Rollback: copy `ca.json.bak-2026-10-03` over `ca.json` and restart the container.

## Open items found
- `secrets/root_ca_key` exists in the container. The real root key is supposed to be offline on [[pve-ca-root]], so this is probably a leftover from the throwaway bootstrap PKI. Not verified.
- `secrets/password` is mode 644 (readable by any local user in the container).
- Not yet tested: whether sshd accepts an EC P-256 SSH CA key. Fallback is Ed25519 (`--kty OKP --curve Ed25519`).
- Not set: SSH certificate duration claims on the provisioner. Defaults apply until pinned.

## Workstation, Authentik and the first certificate
- `step ca bootstrap --ca-url https://stepca.homelab.local:9000 --fingerprint <root fingerprint>` run on the workstation. The fingerprint came from `docker exec step-ca step certificate fingerprint /home/step/certs/root_ca.crt`. The vault holds no stored fingerprint value to compare against, only the note that it was verified against [[pve-ca-root]] earlier.
- First `step ssh certificate` attempt failed in Authentik with "Redirect URI Error". Cause: the step CLI sends `http://127.0.0.1:<random port>` as the redirect URI, and the Authentik provider `step-ca` only listed `https://stepca.homelab.local:9000/`. Fix: added a second Redirect URIs entry in the provider, mode Regex, type Authorization, value `http://127\.0\.0\.1:\d+/?`. The first save kept the mode as Strict (the overview showed `strict:` in front of it), so it had no effect until the mode was changed to Regex.
- Next failure: `invalid_client` on the token exchange. The step-ca provisioner secret did not work against the Authentik provider. Re-entered the provider's client secret into the provisioner with `step ca provisioner update Authentik --client-secret ...` using the `step` admin and the `admin` JWK provisioner, with the secret typed at a masked prompt so it stayed out of chat and shell history. The token exchange worked afterward. I did not find out what the old stored secret was. `step ca provisioner list` returned the `clientSecret` field as an array of 2 items, which I could not explain.
- The Authentik `step-ca` provider is assigned to its own application `step-ca`. It is not the `cloudflare-access` application used for WARP enrollment.

## What the certificate contains
- `step ssh certificate` with `--no-agent --no-password --insecure` gave an ECDSA user certificate. Key ID `user@example.com`, principals `[cf-team]` and `user@example.com`, no critical options, valid about 16 hours (the provisioner default, not pinned yet).
- Tested whether the client controls the principals: second certificate with identity label `user@example.com` and `--principal [admin-user]`. The result was identical to the first: same Key ID and principals, no `[admin-user]`, no `notme`. So step-ca takes both from the Authentik login and ignores the request. The OIDC login for this account therefore always yields those two principals.
- MFA: the Authentik login for step-ca went straight through on the existing Authentik session, with no MFA prompt (answered by Daniel). The Sep 20 warning is real. WARP login and SSH certificate login are currently one gate, not two. Not fixed. See [[bastion-ssh-certificate-login]].

## sshd on the bastion
- Daniel chose `[admin-user]` as the account for certificate logins.
- On [[pve-bastion]], as root through `pct enter 2202`:
  - `/etc/ssh/step_user_ca.pub` holds the SSH user CA public key. Its fingerprint `SHA256:n+27525/23Nh1Dv4vV1AMNc9NI9E0TtbnQ1zXpHUAnU` matched the "Signing CA" fingerprint on the test certificate.
  - `/etc/ssh/auth_principals/[admin-user]` holds one line, `user@example.com`. The full email was chosen over the short name `[cf-team]` because the short name is only the part before the at sign and could collide with another Authentik user at a different domain.
  - `/etc/ssh/sshd_config.d/10-step-ca.conf` sets `TrustedUserCAKeys` and `AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u`.
  - `/etc/ssh/sshd_config.d/20-no-passwords.conf` sets `PasswordAuthentication no` and `KbdInteractiveAuthentication no`. `sshd -T` confirmed both read `no` and `usepam yes`, `pubkeyauthentication yes`.
- The main `sshd_config` has `Include /etc/ssh/sshd_config.d/*.conf` at line 12 and set none of these options, so the drop-ins apply.
- Tests from the workstation over the Mesh: certificate only login (`PreferredAuthentications=publickey`, key used nowhere else) gave a `[admin-user]` shell before and after the final restart. A password only attempt returns `Permission denied (publickey)` with no prompt.
- Way back in if certificates stop working: `sudo pct enter 2202` from [[pve-services]], delete `20-no-passwords.conf`, `systemctl restart ssh`.

## What went wrong (for next time)
- `systemctl reload ssh` on the bastion made sshd exit with `fatal: Cannot bind any address`. Debian 13 starts sshd through `ssh.socket`, so systemd already holds port 22 and the re-executed sshd cannot bind it. The unit showed `failed` while systemd kept listening. `systemctl restart ssh` fixed it. Use restart on this host, never reload. This is my reading of the journal, not something I found in the Debian docs.
- I misread two PowerShell outputs about the provisioner secret (a length of 2 against a long printed value) and did not understand the `Object[]` result. The secret was re-entered rather than diagnosed.
- A Regex entry saved as Strict in Authentik. Always read the overview line afterward: it shows `regex:` or `strict:`.
- Daniel's step-ca admin password is in plain text in a Sep 23 chat. It is also stored in Vaultwarden. Rotate it.

## Left open
- Temporary `workstation` to `bastion` port 22 rule in OPNsense: Daniel reported it deleted at about 15:12 local on Oct 3, 2026. Which interface tab it was on, and whether Apply Changes was run, were not confirmed in chat, and no test of direct LAN SSH to 10.0.51.x was run afterward.
- MFA gap: the step-ca login reuses the Authentik session. Decide how to force a fresh login.
- Pin the SSH user certificate lifetime on the Authentik provisioner (min, default and max). Currently about 16 hours.
- The new SSH CA private keys exist only in the step-ca Docker volume. Backup of that volume is not verified.
- Throwaway test keys `ssh-test-key` and `ssh-test-key2` (with `-cert.pub` and `.pub`) are still in `.step` on the workstation. Delete them.
- Rotate the step-ca admin password. Check `secrets/root_ca_key` and the mode 644 on `secrets/password` inside the step-ca container.
- SSH certificate login uses a key made by `step ssh certificate`. The normal flow for daily use (`step ssh login` with the Windows ssh-agent) was not tested.

---
name: bastion-ssh-certificate-login
type: decision
status: live
---

## Decision
[[pve-bastion]] accepts SSH only with a certificate signed by the step-ca SSH user CA ([[pve-int-stepca]]). Password and keyboard-interactive login are off. Built and tested Oct 3, 2026, see [[session-2026-10-03-stepca-ssh-ca]].

- One Linux account takes certificate logins: `[admin-user]` (chosen by Daniel).
- The certificate principal that may log in as `[admin-user]` is the full email `user@example.com`, listed in `/etc/ssh/auth_principals/[admin-user]`.
- Config lives in two drop-ins under `/etc/ssh/sshd_config.d/`: `10-step-ca.conf` and `20-no-passwords.conf`. The main `sshd_config` is unchanged.

## Why
- step-ca issues the principals itself from the Authentik login and ignores the client's request, tested Oct 3, 2026 with a different identity label and a requested `[admin-user]` principal. So the principals are not client controlled.
- The full email is a tighter match than `[cf-team]`, which is only the part before the at sign.

## Ruled out
- A new dedicated account: more to maintain for one admin. The opposing view was that it keeps the password account and the certificate account separate. With passwords off, that point is small.
- Matching on the short name `[cf-team]`: it can collide with another user whose email has the same local part.

## Known weakness, fixed Oct 17, 2026
Fixed: the `step-ca` provider's Authorization flow is now `step-ca-authorization-mfa`, which runs the Authenticator Validation stage `step-ca-mfa-every-time` (TOTP only, Deny if not configured, Last validation threshold `seconds=0`). Tested twice back to back while logged in to Authentik: TOTP asked each time, certificate written each time. WARP enrollment and the SSH certificate are now two separate MFA events. Rollback: provider back to `default-provider-authorization-implicit-consent`. See [[session-2026-10-17-stepca-remote-tunnel]].

Original note:
The step-ca login reused an existing Authentik session with no MFA prompt (Oct 3, 2026). The Sep 20 note warned about this. WARP enrollment and the SSH certificate currently count as one authentication event. Options not yet chosen: a re-authentication stage or a short session in Authentik for the `step-ca` application, or a short certificate lifetime on the provisioner so a fresh login is needed often. The Authentik screens for this were not checked.

## Rules that must stay true
- On the bastion, use `systemctl restart ssh`, never `reload`. sshd is started by `ssh.socket` and a reload exits with `Cannot bind any address`.
- Never stop or restart `nftables.service` on the bastion ([[bastion-host-firewall]]).
- Way back in without a certificate: `sudo pct enter 2202` from [[pve-services]].

## Related
- [[ssh-bastion-certificate-login]]
- [[pve-bastion]]
- [[pve-int-stepca]]
- [[authentik]]
- [[sop-ssh-bastion-build|SSH Bastion Build]]

---
name: vaultwarden-harden-container
type: runbook
tool: vaultwarden
---

Checklist for hardening the Vaultwarden container on [[services-host]]. Based on the official hardening guide plus findings from this homelab. Mark each item as it is confirmed on the real container. Nothing is assumed done.

## From the official guide
- [ ] Disable open registration. See [[vaultwarden-disable-signups-and-invite-users]].
- [ ] Disable invitations by registered users (the `INVITATIONS_ALLOWED` setting, or the matching admin page toggle).
- [ ] Hide password hints (the admin page option "Show password hints" unchecked).
- [ ] HTTPS through a reverse proxy, not Vaultwarden's built in TLS (the docs say Rocket's TLS is not production ready). See [[vaultwarden-npm-proxy-host]].
- [ ] Strict SNI on the reverse proxy, so the vault cannot be reached by bare IP.
- [ ] Redact `access_token` values in proxy access logs and anything shipped to external log services.
- [ ] Run as a non-root user (`user: 1000:1000` in compose, or `-u 1000:1000`). The data folder must be owned by that UID first, or the container cannot write.
- [ ] Mount only what is needed. Add `:ro` to any mount that does not need writes.
- [ ] Fail2ban. See [[vaultwarden-configure-fail2ban]].
- [ ] Optional: serve from an obscure sub path.

## From this homelab
- [ ] Two factor on every account. TOTP is on for the main account, see [[vaultwarden-enable-mfa]].
- [ ] Admin page off, or protected by a hashed token. See [[vaultwarden-enable-admin-page]].
- [ ] The published port `8080:80` binds to all interfaces of [[services-host]] by default, so anything that can reach the host IP can bypass the proxy. A Sep 2026 session flagged this pattern for the host. If NPM reaches Vaultwarden over `reverse-proxy-net`, the publish may not be needed, but confirm that before removing it. Otherwise rely on the OPNsense rules for the Services VLAN.
- [ ] Docker socket not mounted into the Vaultwarden container (check with `docker inspect`).
- [ ] Wazuh file integrity monitoring on the data folder. Suggested in the Sep 2026 hardening review, status not confirmed.
- [ ] Offline backup working. See [[vaultwarden-schedule-offline-backup]].
- [ ] Container kept current. See [[vaultwarden-update-container]].

## Related
- [[vaultwarden]]
- [[services-host]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Hardening-Guide

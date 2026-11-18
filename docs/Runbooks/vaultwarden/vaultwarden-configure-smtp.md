---
name: vaultwarden-configure-smtp
type: runbook
tool: vaultwarden
---

Give Vaultwarden outbound email, which is needed for invitation emails, email verification and some account notices. SMTP state on [[services-host]] is <FILL IN: not recorded>.

## Settings (names from the official wiki)
- `SMTP_HOST`
- `SMTP_FROM`
- `SMTP_PORT`
- `SMTP_SECURITY`
- `SMTP_USERNAME`
- `SMTP_PASSWORD`
- `SMTP_AUTH_MECHANISM`
- `DOMAIN` (used to build the links in invitation emails, must be your real https address)
- `HELO_NAME` (optional, overrides the machine hostname)

## SMTP_SECURITY values
- `starttls`: the default. Port defaults to 587. Only TLS 1.1 and 1.2 are accepted.
- `force_tls`: implicit TLS, typically port 465.
- `off`: port defaults to 25. The docs call this potentially very insecure. Do not use it over an untrusted path.

## Steps
1. Choose a mail provider or relay: <FILL IN>. Many ISPs block outbound port 25.
2. Create a dedicated sending account, not your personal mailbox, and generate an app password if the provider offers one.
3. Add the settings above to the container environment. Keep `SMTP_PASSWORD` in a `.env` file with tight permissions, not in a compose file that is committed anywhere.
4. Special characters in a password need care: backslashes must be doubled, and single quotes escaped when the value is wrapped in single quotes.
5. Re-create the container, then send a test invitation or use the admin page mail test if your version has one. The docs page did not describe an admin test, so verify on your version.
6. Store the SMTP account in Vaultwarden under `homelab/apps` or `homelab/cloud`.

## Gotchas
- Microsoft and Gmail can fail with "No compatible authentication mechanism was found", because they now want OAuth2, which Vaultwarden does not support natively.
- Open items from the Oct 3 pass: whether the Vaultwarden mail account exists at all is unknown.

## Related
- [[vaultwarden-disable-signups-and-invite-users]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration

---
name: vaultwarden-disable-signups-and-invite-users
type: runbook
tool: vaultwarden
---

Close open registration on Vaultwarden and add a new user the controlled way, through an admin invitation. Current signup state on [[services-host]] is <FILL IN: unknown until checked>. Organizations and shared collections are not in use, so for now this is mainly about keeping strangers out.

## Close signups

1. Set the environment variable on the container:
   ```
   SIGNUPS_ALLOWED=false
   ```
   In compose this is a line under `environment:` as `SIGNUPS_ALLOWED: "false"`. The official compose example even advises switching it to false after you create your own account.

2. Optional: allow only your own email domain with `SIGNUPS_DOMAINS_WHITELIST=<domain>`. When this is set, `SIGNUPS_ALLOWED` is ignored.

3. Optional: require email verification with `SIGNUPS_VERIFY=true`. Needs working SMTP, see [[vaultwarden-configure-smtp]].

4. Re-create the container and check that the web vault no longer offers account creation.

## Invite someone (works even with signups off)

1. The admin can invite anyone from the admin page, regardless of the signup restrictions. Admin page setup: [[vaultwarden-enable-admin-page]].
2. Admin page, Users, invite by email address.
3. The invited person registers at `https://<vaultwarden address>/#/signup` using that exact email address. Invitation links are valid for 5 days, per the SMTP docs.
4. After they register, make them enable two-step login ([[vaultwarden-enable-mfa]]).

## Related
- [[vaultwarden-harden-container]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Disable-registration-of-new-users

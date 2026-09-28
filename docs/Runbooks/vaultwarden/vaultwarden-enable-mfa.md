---
name: vaultwarden-enable-mfa
type: runbook
tool: vaultwarden
---

Enable TOTP two-factor auth on [[services-host]]'s Vaultwarden instance. Done Sep 2026.

1. Log into Vaultwarden's web vault, go to account settings, find the Two-Step Login / TOTP section.
2. Select TOTP, scan the QR code with an authenticator app, enter the generated code to confirm.
3. **Save the recovery codes somewhere outside Vaultwarden itself.** This is the step that actually matters: a password manager's own recovery codes can't live inside that same password manager, if the authenticator app is ever lost, these codes are the only way back in without admin-level database access. Write them down physically or store them in a genuinely separate location.
4. Log out and back in to confirm the TOTP prompt appears and a valid code from the app works.

## Related
- [[services-host]]

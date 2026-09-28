---
name: gitea-enable-mfa
type: runbook
tool: gitea
---

Enable TOTP two-factor auth on [[services-host]]'s Gitea instance. Done Sep 2026.

1. Log into Gitea, profile icon top right → Settings → Security tab.
2. Under Two-Factor Authentication, set up TOTP: scan the QR code (same authenticator app as [[vaultwarden-enable-mfa|Vaultwarden]] works fine, just a separate entry), enter the generated code to confirm.
3. Save the recovery codes somewhere outside Gitea, not as a note stored in Gitea, and not inside Vaultwarden either since Vaultwarden being down is exactly the scenario recovery codes are meant to cover.
4. Log out and back in to confirm the TOTP prompt appears and a valid code works.

## Important caveat

**This only protects the Gitea web UI.** Gitea's SSH access (port 222, key-based auth for git push/pull) is a completely separate path, unaffected by TOTP. Don't assume 2FA here extends to SSH pushes/pulls, it doesn't, key security is what protects that path instead.

## Related
- [[services-host]]
- [[vaultwarden-enable-mfa]]

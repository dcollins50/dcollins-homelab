---
name: vaultwarden-configure-sso-authentik
type: runbook
tool: vaultwarden
---

Let users sign in to Vaultwarden through [[authentik]] (OpenID Connect). This is the Authentik SSO item from the Oct 3, 2026 scope in [[vaultwarden-integration]]. Not built yet.

## Read this first
- The official wiki says a master password is still required and is not controlled by SSO. SSO changes how you authenticate to the server, it does not remove the encryption passphrase.
- Signup is blocked if the identity provider reports the email as unverified.
- Which Vaultwarden release added SSO: <FILL IN: check the release notes, not confirmed during this write-up>. Update first if your version lacks it, see [[vaultwarden-update-container]].
- Take a backup first: [[vaultwarden-backup-data]].

## Steps

### 1. In Authentik
Follow [[authentik-oidc-external-service]], with these choices:
- New dedicated OAuth2/OpenID provider named `vaultwarden`. Do not reuse another one.
- Client type: Confidential.
- Redirect URI, exactly: `https://<vaultwarden address>/identity/connect/oidc-signin`. Vaultwarden builds this from its `DOMAIN` setting, so `DOMAIN` must be correct first.
- Create the application with "with Existing Provider" and note its slug.
- Copy the client id and secret fully. If the secret looks truncated, pull it with `ak shell` as in that runbook.
- Authentik's guidance in the Vaultwarden wiki: make the access token lifetime longer than the default 5 minutes, otherwise Bitwarden clients can misread expiry.

### 2. In Vaultwarden
Add the settings (names from the wiki):
- `SSO_ENABLED=true`
- `SSO_AUTHORITY=https://<authentik host>/application/o/<slug>/` (the trailing slash is required)
- `SSO_CLIENT_ID=<client id>`
- `SSO_CLIENT_SECRET=<client secret>` (keep it in a `.env` file, never in a note)
- `SSO_SCOPES=openid profile email offline_access` (Authentik example from the wiki)
- Optional later: `SSO_ONLY=true` disables email and password sign in. Do not enable it until SSO is proven to work, and keep a break glass path, see [[vaultwarden-recover-access]].
- PKCE is on by default (`SSO_PKCE`).

### 3. Check the network path
Vaultwarden's backend calls Authentik directly, not only the browser. [[services-host]] is on VLAN20 and [[authentik]] is on VLAN30 (10.0.30.x), so check the OPNsense rule from VLAN20 to Authentik, and that the Authentik hostname resolves from the container. A working browser redirect does not prove this. Same lesson as the existing OIDC runbook.

### 4. Test
1. Restart the container.
2. Open the web vault in a private window and choose Enterprise single sign-on, or the SSO option your version shows.
3. Log in through Authentik, set the master password when asked, and confirm the vault opens.
4. Confirm MFA still applies. Authentik enforces TOTP, see [[authentik-enforce-mfa]].

## If it fails
Use [[authentik-diagnose-flow-failures]] for the Authentik side, and `docker logs <container>` for the Vaultwarden side.

## Related
- [[authentik]]
- [[authentik-oidc-external-service]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect

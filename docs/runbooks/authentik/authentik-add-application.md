# Runbook: Add an Application (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Identity / SSO |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a new service needs to sit behind Authentik, whether it natively supports OIDC/SAML (rare for small self-hosted apps) or has no login of its own and needs forward-auth protection at the reverse proxy layer (the common case here, e.g. ntfy).

---

## Before you start

- **Decide the provider type.** For an app with no native auth support, that's a **Proxy provider**. For an app that speaks OIDC/SAML itself, use the matching native provider type instead, this runbook focuses on the Proxy provider case since that's this environment's actual recurring pattern.
- **Decide the proxy mode:**
  | Mode | Use when |
  |---|---|
  | Proxy | Authentik's own outpost should proxy traffic to the app directly |
  | Forward auth (single application) | Your existing reverse proxy (e.g. NPM) handles the app's traffic and only asks Authentik to check auth. Each app gets its own provider. **This is the pattern already in use here (ntfy).** |
  | Forward auth (domain level) | One provider covers multiple apps under the same parent domain. Fewer providers to manage, but you lose the ability to set different access policies per app, don't use this for anything that needs its own access rules. |
- **Have an outpost ready**, or plan to create one as part of this, see [Configure an Outpost](authentik-configure-outpost.md).

---

## Steps

### 1. Create the application and provider together

Applications → Applications → **New Application**. Creating both together (rather than the legacy separate-provider path) is the recommended flow for the common case.

- **Name / Slug:** descriptive, matches the app
- **Provider:** choose to create a new one inline

### 2. Configure the Proxy provider

- **Mode:** per the table above
- **External host:** the URL users actually use to reach the app (scheme + domain, and port if non-standard)
- **Internal host:** the upstream URL the outpost forwards to (only relevant in Proxy mode, not forward-auth, since forward-auth traffic never routes through Authentik itself)
- **Internal host SSL Validation:** whether to validate the upstream's TLS cert, matters if the upstream uses a self-signed or internally-issued cert
- For **forward auth (domain level)** specifically: set **Authentication URL** (the external auth-check URL) and **Cookie domain** (the shared parent domain across protected apps)

### 3. Assign the application to an outpost

See [Configure an Outpost](authentik-configure-outpost.md) if one doesn't already exist for this app. Without this step, the provider exists but nothing enforces it.

### 4. Configure the reverse proxy (forward-auth modes only)

On the actual reverse proxy in front of the app (NPM, in this environment's case), add the auth-check configuration Authentik's UI provides for the provider (an `auth_request` block pointing at the outpost's forward-auth endpoint, plus the required headers). The exact template differs by proxy software, use the configuration snippet Authentik generates for the specific provider rather than a generic one, since it's tied to that provider's slug/URL.

### 5. Configure access control (optional but recommended)

By default, **an application with no bindings is open to every Authentik user.** If this app shouldn't be available to everyone, bind a group, user, or policy to it now: open the application → **Policy / Group / User Bindings** tab → Create or bind. See [Manage Permissions](authentik-manage-permissions.md) for the fuller picture on groups vs. policies here.

### 6. Verify

- Visit the app's external URL in a private/incognito window
- Confirm you're redirected to Authentik's login
- Confirm that after login you land back on the app, and that it received the expected identity headers if the app itself reads them (`X-authentik-username`, `X-authentik-email`, etc.)
- If access bindings were set in step 5, confirm they're actually enforced, test with an account that should be denied, not just one that should be allowed

---

## Common mistakes

- **Choosing domain-level forward auth for an app that will eventually need its own access policy.** Migrating off domain-level later means creating a dedicated provider anyway, decide up front if this app is genuinely part of a shared-policy group or not.
- **Forgetting step 3 (outpost assignment).** The most common reason a freshly configured provider does nothing.
- **Assuming an app is protected because a provider exists**, without checking whether any access binding was actually set, remember the default is open to everyone.
- **Copying a generic reverse-proxy auth snippet instead of the provider-specific one Authentik generates**, small mismatches in the auth endpoint path or header names cause confusing failures.

---

## Related Documentation

- [Configure an Outpost](authentik-configure-outpost.md)
- [Manage Permissions](authentik-manage-permissions.md)
- [Internal PKI](../../pki.md)

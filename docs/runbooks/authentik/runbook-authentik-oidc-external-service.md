# Runbook: Add a Dedicated Authentik OIDC Provider for an External Service

**Category:** Authentik
**When to use:** Connecting a new external system (Cloudflare Access, step-ca, or similar) to Authentik as its identity provider via OpenID Connect.

## Steps

1. **Create a dedicated provider — do not reuse an existing one across unrelated integrations.** Even if two integrations both need "an OIDC provider," give each its own, named for what it's actually for (e.g. `cloudflare-access`, `step-ca`). This keeps client secrets scoped, makes revocation clean, and avoids one integration's redirect URI config leaking into another's.

   **Applications → Providers → Create**
   - Type: OAuth2/OpenID Provider
   - Name: descriptive, matching the consuming system
   - Authorization flow: your org's standard flow for this trust level (e.g. implicit consent for internally-trusted apps)
   - Client Type: Confidential
   - Redirect URI: the exact callback URL the consuming system expects (get this from that system's own setup docs/UI — don't guess the path)
   - Save, then **immediately copy the full Client ID and Client Secret** — the secret is truncated in list views and edit forms alike; if in doubt, pull it directly from the object rather than trusting what's visible in the UI:
     ```python
     docker exec -it <authentik-worker-container> ak shell
     from authentik.providers.oauth2.models import OAuth2Provider
     p = OAuth2Provider.objects.get(name="<provider-name>")
     print(p.client_id, p.client_secret, p.redirect_uris, p.client_type)
     ```

2. **Create the matching application, using "with Existing Provider," not the default wizard.** Authentik's default "New Application" flow bundles creating a *new* provider alongside the application — if you already built the provider in step 1, this creates an unwanted duplicate. Look for an explicit "with Existing Provider" option, or create the application separately and link the provider via its dropdown.
   - Note the application's **slug** — this becomes part of the OIDC discovery URL: `https://<authentik-host>/application/o/<slug>/.well-known/openid-configuration`.

3. **Pull the discovery document to get the exact endpoint URLs** the consuming system will need (many systems, like Cloudflare Access's generic OIDC form, want Auth/Token/Certificate URLs as separate fields, not the one discovery URL):
   ```bash
   curl -s https://<authentik-host>/application/o/<slug>/.well-known/openid-configuration
   ```
   Pull `authorization_endpoint`, `token_endpoint`, and `jwks_uri` from the response.

4. **Confirm Authentik itself is actually reachable from wherever the consuming system's backend runs.** This is the step most likely to be silently assumed rather than checked. A browser-based login redirect succeeding is not proof the *server-to-server* token exchange will work — many OIDC flows (Cloudflare Access included) have the relying party's own backend call the token endpoint directly, which means Authentik's hostname needs to resolve and be reachable from wherever that backend actually lives (often the public internet, not just your LAN).

5. **Test the connection using whatever built-in test the consuming system offers** before moving on to the next stage of setup. Don't assume a successful provider save means the integration works end to end — test the actual token exchange.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| Consuming system's test says "Failed to exchange code for token, check client secret" | Client secret was copied truncated from a UI field | Pull the exact value via `ak shell` (step 1) and re-enter it fresh |
| Token exchange fails with no clear error, discovery URL works fine from a browser | Authentik's hostname isn't actually reachable from the consuming system's backend (only resolves on your internal network) | Confirm public reachability from a genuinely external network before assuming the OIDC config is wrong |
| Duplicate/unused OAuth2 provider objects accumulating | Used the default "New Application" wizard after already creating a provider separately | Use "with Existing Provider" going forward; clean up unused providers if found |
| A consuming system (e.g. Cloudflare Access) accepts login but nothing beyond it ever succeeds, with the IdP test passing | The consuming system requires its own separate enrollment/access policy in addition to the IdP integration itself | Check the consuming system's own docs for an "enrollment permissions" or "access policy" requirement — an IdP connection and a policy allowing its use are often two distinct configuration steps |

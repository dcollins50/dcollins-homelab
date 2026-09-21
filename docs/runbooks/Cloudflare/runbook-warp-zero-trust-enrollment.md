# Runbook: Cloudflare WARP Install + Zero Trust Device Enrollment

**Category:** Cloudflare Tunnel / Zero Trust
**When to use:** Setting up a headless Linux host to enroll as a managed device in a Cloudflare Zero Trust organization, authenticating through a third-party IdP (Authentik or similar) rather than a bare one-time PIN.

**Status as of last use:** Setup steps below are confirmed correct and complete through device enrollment permissions. Final enrollment click was blocked by a browser-side `ERR_QUIC_PROTOCOL_ERROR` on the authorize redirect — see Known Issue at the bottom before assuming a fresh attempt will fail the same way you configured something wrong.

## Prerequisites

- A Cloudflare Zero Trust organization with a team name set (Settings → General — rename from the auto-generated default if desired; note this requires updating IdP config and every enrolled device if changed later).
- An Authentik (or other) OIDC identity provider **already confirmed reachable from the public internet** — Cloudflare Access's backend does the token exchange server-side, not just the initial browser redirect, so this needs real public reachability, not just internal DNS. See the Cloudflare Tunnel publish runbook if this isn't already true.

## Steps

**1. Install the WARP package** (this pulls in the full GUI client dependency chain even on a headless box — expected, not a mistake):
```bash
mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/cloudflare-client.list
apt update
apt install cloudflare-warp -y
```

**2. Build a dedicated OIDC identity provider integration** for Cloudflare Access itself, separate from any other OIDC integration the IdP already has (e.g. don't reuse a provider built for a different consuming system). See the Authentik OIDC runbook for the general provider/application creation steps.

**3. Add the IdP under Zero Trust → Settings → Authentication (or Team & Resources → Integrations → Identity providers) → Add an identity provider → OpenID Connect.** Fill in App ID (Client ID), Client secret, and the Auth/Token/Certificate URLs pulled from the IdP's discovery document. Test the connection here before moving on — Zero Trust's own "Test" link confirms the OIDC handshake works, independent of device enrollment.

**4. Configure Device enrollment permissions — do not skip this.** This is a separate requirement from having an IdP integration configured, and its absence produces a generic, misleading "Enrollment request is invalid" error that looks like a broken IdP config when it's actually just a missing policy.

   Team & Resources → Devices → Management → Device enrollment (Manage):
   - **Policies tab**: create at least one Allow policy (e.g. Include → Emails → specific address, or a domain-wide rule)
   - **Login methods / Authentication tab**: leave "Accept all available identity providers" on unless you specifically want to restrict enrollment to one IdP only

**5. Enroll the device.** Run as a non-root user (WARP explicitly refuses to run as root):
```bash
su - <user>
warp-cli registration new <team-name>
```
Accept the ToS prompt, then **immediately** open the printed URL in a browser — the enrollment session is short-lived; navigating to it later reliably produces "Enrollment request is invalid" that looks configuration-related but is actually just a stale session.

**6. Verify:**
```bash
warp-cli registration show
warp-cli registration organization
```

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| `warp-cli teams-enroll` — unrecognized subcommand | Deprecated in current client versions | Use `warp-cli registration new <team-name>` instead |
| "Enrollment request is invalid" on the browser page | Either a stale/reused enrollment URL, or (far more likely if it happens on every fresh attempt) Device enrollment permissions was never configured | Retry with a genuinely fresh `registration new` run; if it still fails every time, check step 4 above first |
| IdP's own "Test" link passes, but device enrollment still fails | Confirms the OIDC connection itself is fine — the problem is downstream, in device enrollment permissions or an access policy, not the IdP config | Re-check step 4 |
| Identification step (email/username prompt) fails with "invalid identifier" in the IdP's own logs | Wrong value typed (e.g. a username fragment instead of the full email the account actually uses) | Confirm the exact identifier value the account is provisioned under |
| Identification succeeds (confirmed via IdP server logs — no error at that stage) but the browser never shows a password/next-stage prompt | Client-side failure after a successful server response — check IdP flow stage bindings are intact, and rule out corrupted session cookies with a private/incognito window | If a private window changes nothing, the issue is likely below the application layer (see Known Issue) |

## Known Issue (unresolved as of last session)

`ERR_QUIC_PROTOCOL_ERROR` on the `/application/o/authorize/` redirect specifically, reproduced across multiple browsers and networks, with server-side logs showing no error at all for the corresponding request (meaning it may not be reaching the server in a form the server can log). The authorize URL carries a large base64-encoded `state` parameter (several KB) — QUIC/HTTP3 has stricter size limits than HTTP/1.1 or HTTP/2, and this is the leading unconfirmed theory. Next diagnostic step: test the exact failing URL with `curl` (which won't attempt QUIC) to confirm whether the request itself succeeds over HTTP/2, which would isolate this to protocol negotiation rather than anything in the Access/IdP/tunnel configuration.

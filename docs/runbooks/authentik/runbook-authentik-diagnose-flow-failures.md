# Runbook: Diagnose Authentik Authentication Flow Failures via Server Logs

**Category:** Authentik
**When to use:** A login/OIDC/enrollment flow through Authentik fails with a vague or generic error on the consuming application's side (e.g. "invalid request," "failed to get identity"), and you need to see what Authentik itself actually did with the request.

## Finding the right container

Authentik typically runs as multiple containers (worker, server, and possibly others). Log activity relevant to live HTTP requests and login attempts lives on the **server** container, not the worker:
```bash
docker ps
docker logs <authentik-server-container> --tail 50
```

## Getting logs precisely timed to a reproduction

A long `--tail` pulls in a lot of routine noise (health checks, periodic outpost refreshes). Two better approaches:

**Reproduce the failure, then immediately pull a tight time window:**
```bash
docker logs <authentik-server-container> --since 2m
```

**Or grep for a specific stage/endpoint while tailing more broadly:**
```bash
docker logs <authentik-server-container> --tail 200 | grep "token"
```

## Reading the output

Authentik logs structured JSON, one event per line. Useful fields:
- `"event"` — what happened (e.g. `invalid_login`, `authorize_application`, or a raw request path)
- `"action"` — for policy/flow events specifically (e.g. `invalid_identifier`, `model_created`)
- `"identifier"` — for login attempts, the exact value the user submitted (useful for confirming *what* was actually typed, not what you assume was typed)
- `"status"` — HTTP status code for request-log lines
- A request that reaches Authentik but the flow doesn't progress further (no error, no next-stage log entry) suggests the break is happening **client-side**, after Authentik's response but before the browser acts on it — not a server-side rejection. Rule out browser state next (private/incognito window; disabling HTTP/3-QUIC in browser flags if requests to this host are failing with a QUIC-specific browser error) before continuing to chase server-side config.

## Deeper checks via the admin shell

For a Django-app-based system like Authentik, direct model inspection is often faster than guessing from logs alone. Enter the worker container's shell:
```bash
docker exec -it <authentik-worker-container> ak shell
```

**Check a provider's exact stored credentials** (never trust a truncated value from a UI field):
```python
from authentik.providers.oauth2.models import OAuth2Provider
p = OAuth2Provider.objects.get(name="<provider-name>")
print(p.client_id, p.client_secret, p.redirect_uris, p.client_type)
```

**Check whether a reputation policy is blocking a user/IP** (failed login attempts dock a score; a sufficiently negative score silently blocks further attempts):
```python
from authentik.policies.reputation.models import Reputation
for r in Reputation.objects.all():
    print(r.identifier, r.ip, r.score)
```

**Confirm a flow's stage sequence is actually intact** (a missing or misconfigured stage can cause a flow to silently fail to progress):
```python
from authentik.flows.models import Flow, FlowStageBinding
flow = Flow.objects.get(slug="<flow-slug>")
for binding in FlowStageBinding.objects.filter(target=flow).order_by('order'):
    print(binding.order, binding.stage.name, binding.stage)
```

## Notes

- A test that passes at the IdP integration level (e.g. Cloudflare Access's own "Test" button for an identity provider) confirms the OIDC handshake itself works — it does **not** confirm anything about enrollment policies, access policies, or other gating configuration the consuming system may separately require. Don't stop investigating just because that one test is green.

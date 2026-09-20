# Runbook: Configure an Outpost (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Identity / SSO |
| Author | Daniel Collins |

---

## When to use this

An outpost is what actually enforces authentication for a Proxy provider, it's the component that either proxies traffic to the protected app itself, or answers the "is this request authenticated?" check your reverse proxy sends it (forward auth). Every proxy-mode application needs to be assigned to one. Use this runbook when setting up the outpost for a new proxy application, or troubleshooting why an app's auth checks aren't being answered.

---

## Outpost types

- **Embedded outpost** — runs as part of the Authentik server itself, no separate deployment. Simplest option, and the right default when Authentik is already sitting behind a reverse proxy and you want proxy-provider traffic handled by that same deployment. Accessible on the same ports as Authentik itself (9000 HTTP / 9443 HTTPS).
- **Managed outpost** — a separate outpost Authentik deploys and manages itself via a Docker or Kubernetes integration, with its own lifecycle independent of the core server.
- **Manually deployed outpost** — you deploy and manage the outpost container yourself, Authentik just issues it credentials.

For a single-node homelab deployment where Authentik itself is already reverse-proxied, the embedded outpost is the simplest fit and the default choice unless there's a specific reason to isolate an app's outpost lifecycle from the core server.

---

## Steps

### Creating an outpost

1. **Applications → Outposts → Create.**
2. **Name:** something descriptive of what it's fronting, if it's dedicated to one app or app group.
3. **Type:** Proxy (for proxy-mode or forward-auth applications).
4. **Integration:** select Docker/Kubernetes for a managed outpost, or leave unset for manual/embedded.
5. **Applications:** select which application(s)/provider(s) this outpost should serve.
6. **Create.** Authentik auto-generates a service account and token scoped only to the assigned application/provider objects, this is what the outpost uses to authenticate to the Authentik API.

### Using the embedded outpost instead

If you don't need a separate outpost lifecycle, skip creating a new one and assign the application's provider to the existing embedded outpost (**Applications → Outposts**, select the embedded outpost, add the provider to its Applications list). This is the pattern already in use for this environment's forward-auth setups.

### Verifying an outpost is healthy

1. Applications → Outposts, check the outpost's status indicator.
2. Dashboards → System Tasks, for background task/deployment status on managed outposts.
3. Confirm the outpost's assigned provider(s) actually show up under it, a provider created but never assigned to any outpost will never enforce anything.

---

## Common mistakes

- **Creating the provider/application but forgetting to assign it to an outpost.** The provider exists but nothing is actually enforcing auth for it, the app remains unprotected.
- **Spinning up a new managed outpost when the embedded one would do.** Adds deployment complexity without a real benefit for a small, single-node setup.
- **Not checking the outpost's health after adding a new application to it.** A change should immediately propagate (Authentik pushes config updates to outposts over websockets), but it's still worth confirming rather than assuming.

---

## Related Documentation

- [Add an Application](authentik-add-application.md)
- [Internal PKI](../../pki.md)

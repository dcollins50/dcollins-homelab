# Session Log — September 20, 2026 (AM): SSH Bastion MFA Design

**Scope:** Designing the actual multi-layer authentication architecture for the SSH Bastion, building on the path decision from [Session Log — September 18, 2026](session-log-sept18-bastion-path.md).

---

## Requirement

At least **two discrete MFA events** in the authentication flow required to reach the bastion, not one MFA check that happens to gate two different technical steps.

## The Naive Approach Doesn't Actually Satisfy the Requirement

A straightforward Cloudflare Access + Authentik IdP setup produces only **one logical MFA event**: Authentik acting as the OIDC/SAML IdP behind Access is a single authentication, however many technical hops it spans. Recognizing this before building it out avoided ending up with something that looked like two-factor on paper but wasn't in practice.

## Agreed Design

Two genuinely independent MFA events:

1. **MFA event one — WARP client enrollment.** Authentik acts as the Zero Trust IdP for WARP enrollment itself.
2. **MFA event two — step-ca's OIDC provisioner.** Rather than a static SSH key, step-ca issues a short-lived SSH user certificate, and doing so triggers a **fresh** Authentik login (its own OIDC flow, separate from the WARP enrollment session) before it will issue the cert.

The bastion's `sshd` is configured to trust step-ca's SSH user CA key via `TrustedUserCAKeys`, replacing static SSH keys entirely for bastion access.

## The Landmine: Session Reuse

Authentik's default behavior reuses an existing session. Without mitigation, the second OIDC flow (step-ca's provisioner) could silently succeed against the still-active WARP-enrollment session, without re-prompting for MFA at all, collapsing the design back down to one real MFA event despite looking like two on paper.

**Mitigations discussed** (implementation still pending as of this session):
- Scope the step-ca OIDC provisioner to a **dedicated Authentik application** with a short token/session validity, forcing a fresh check rather than reusing the WARP session, or
- Use Authentik's **step-up / re-authentication stage** configuration to force MFA revalidation regardless of existing session state.

## Additional Hardening Noted

- Set explicit WARP enrollment reauthentication intervals, rather than leaving it indefinite.
- Keep SSH certificate TTLs short, since these are meant to be short-lived credentials, not long-standing ones.
- Standard `sshd` lockdown on the bastion, plus Wazuh agent logging on it, same baseline expected of any other hardened host in this environment.

---

## Open items carried forward

- Implementation of the session-reuse mitigation (dedicated Authentik app vs. step-up re-auth stage) is not yet built, this session was design only.
- This depends on step-ca actually being configured, which was deferred as of [Session Log — September 19, 2026](session-log-sept19-stepca-iris-soar.md).

---

## Related Documentation

- [Session Log — September 18, 2026: SSH Bastion Path Decision](session-log-sept18-bastion-path.md)
- [Session Log — September 19, 2026: step-ca / IRIS / Shuffle SOAR](session-log-sept19-stepca-iris-soar.md)
- [Internal PKI](../../pki.md)
- [Network Architecture](../../network.md)

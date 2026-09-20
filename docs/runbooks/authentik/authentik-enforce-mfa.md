# Runbook: Enforce MFA (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Identity / SSO |
| Author | Daniel Collins |

---

## When to use this

Having MFA available to enroll (TOTP is already set up in this environment) is different from MFA being **required**. Nothing forces its use unless an Authenticator Validation stage is actually bound into the authentication flow. Use this runbook when deciding to make MFA mandatory, for everyone or for specific users/groups/applications.

---

## How enforcement actually works

MFA enforcement isn't a single toggle, it's built from flow stages and policies:

- **Authenticator Validation stage** — the stage that actually checks for/prompts an MFA device during a flow. This has to be bound into the authentication flow to have any effect.
- **What happens if the user has no device enrolled yet** is controlled separately, typically via a policy that checks whether the user has a configured authenticator and, if not, routes them to enroll one instead of just skipping the check.
- **Conditional enforcement** (only certain users/groups need to comply) is done via an Expression Policy bound to the validation stage, returning True/False for whether that stage should run for the current user.

---

## Steps

### Enforcing MFA for all users

1. **Flows & Stages → Flows**, open the authentication flow in use (the default login flow, unless a custom one is in play).
2. Add (or confirm already present) an **Authenticator Validation stage** bound into the flow.
3. Bind a policy to the stage that handles the "not yet enrolled" case, deciding whether such users are forced to enroll before continuing, or allowed through unenforced (only appropriate as a temporary grace period, not a permanent state, if the intent is mandatory MFA).
4. Save, then test with a real login, both for an account that already has MFA enrolled, and for one that doesn't, to confirm both paths behave as intended.

### Enforcing MFA only for specific users or groups

1. Same starting point: an Authenticator Validation stage bound into the flow.
2. Instead of applying it universally, bind an **Expression Policy** to the stage's binding that returns `True` (enforce) only for the intended users/groups, and `False` (skip) otherwise. This is Python-based, evaluated against `request.context["pending_user"]` inside authentication flows, don't use `request.user` here, it isn't reliably the user being authenticated at this point in the flow.
3. Test both a user who should be prompted and one who shouldn't.

### Enforcing MFA for a specific application rather than login overall

MFA enforcement described above is a login-flow-level control, it applies at authentication time, not per-application. If the actual need is "this specific app requires stronger assurance than others," that's more naturally handled by which flow the app's provider uses, or by a dedicated flow with its own MFA stage assigned to that provider, rather than trying to bolt app-specific logic onto the shared login flow.

---

## Common mistakes

- **Assuming MFA is enforced because it's available to enroll.** Availability and enforcement are unrelated, an account can have TOTP set up and never be asked for it if no stage actually requires it.
- **Forgetting to handle the "not enrolled yet" case**, a validation stage with no fallback policy can either lock out everyone without a device, or silently pass everyone through, neither of which is usually the intent, decide explicitly which behavior is wanted.
- **Using `request.user` instead of `request.context["pending_user"]`** in an expression policy meant to run during authentication, this is a documented, easy mistake that silently evaluates against the wrong (or no) user.

---

## Related Documentation

- [Manage Permissions](authentik-manage-permissions.md)
- [Manage Users](authentik-manage-users.md)

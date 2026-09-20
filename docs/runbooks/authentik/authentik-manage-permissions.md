# Runbook: Manage Permissions (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Identity / SSO |
| Author | Daniel Collins |

---

## When to use this

This covers two genuinely different questions that are easy to conflate:

1. **"What can this user/group do within Authentik itself, and what system-level rights do they have?"** → Groups, Roles, and object permissions.
2. **"Which users are even allowed to reach this specific application?"** → Policy/Group/User Bindings on the application.

A user can have full admin rights inside Authentik and still be blocked from a specific application if it has a binding that excludes them, and conversely a user with no special permissions at all can still reach any application that has no bindings set, since the default is open. These are separate, independent mechanisms.

---

## Part 1: System permissions (Groups, Roles)

**You cannot grant a permission directly to a user.** Permissions always flow through either:

- **A Group** the user belongs to, with a Role (or roles) attached to that group, or
- **A Role** assigned directly

### Creating a group and granting it permissions

1. **Directory → Groups → Create.**
2. Name the group, set parent group if relevant, and whether it grants superuser status (be deliberate here, superuser is broad, don't grant it as a default).
3. Attach the role(s) that carry the actual permissions this group should have.
4. Save.

### Adding/removing members of a group

1. Directory → Groups, click the group name to open its detail page.
2. Use the **Users** tab to add existing users or create new ones directly into the group.
3. Alternatively, from **Directory → Users → [user] → Groups tab**, add the user to the group from their side, either direction works.

### Managing what a role/group can actually do

1. Open the group's detail page → **Permissions** tab.
2. Grant/adjust object-level or global permissions as needed (e.g. "Can view User" is required for a group to manage other users' details, this isn't implied by other permissions automatically).
3. Note: as of more recent Authentik versions, granting or revoking **superuser** status specifically requires its own explicit permission, separate from general group-edit rights, don't assume the ability to edit a group implies the ability to make it a superuser group.

---

## Part 2: Application access (Bindings)

This is entirely separate from the above. It controls who can even see/launch a given application, regardless of what system permissions they hold.

### How it works

- **By default, an application with no bindings is accessible to every user.** This is not a fail-safe default, restricting access is opt-in, not opt-out.
- A binding attaches a **policy**, a **user**, or a **group** to the application.
- User/group bindings are simple membership checks (pass if the user matches / is a member).
- Policy bindings are for anything conditional (time of day, request context, expression logic), use these when access depends on more than plain membership.
- **Policy engine mode** (Any vs. All) controls how multiple bindings on the same application combine: **Any** passes if at least one binding passes, **All** requires every binding to pass.

### Restricting an application to specific users/groups

1. Open the application (Applications → Applications → [app]).
2. **Policy / Group / User Bindings tab → Create or bind.**
3. Choose **Bind a user** or **Bind a group** (for a straightforward membership rule), or create/bind a policy for conditional logic.
4. Set Policy engine mode if there's more than one binding.
5. Create.
6. **Verify by testing with an account that should be denied**, not just one that should be allowed, a binding that looks correct in the UI can still fail to actually restrict access if misconfigured (this is a real, previously reported failure mode, not hypothetical).

---

## Common mistakes

- **Assuming an application is restricted because a policy/group exists somewhere in Authentik**, without checking that it's actually bound to that specific application.
- **Trying to grant a permission to an individual user directly**, this isn't how Authentik's model works, it always has to go through a group/role.
- **Granting superuser status as a shortcut** to solve a permissions problem instead of scoping the actual role/permission needed.
- **Only testing the "should have access" case**, never confirming that access is actually denied where it should be.

---

## Related Documentation

- [Manage Users](authentik-manage-users.md)
- [Add an Application](authentik-add-application.md)
- [Enforce MFA](authentik-enforce-mfa.md)

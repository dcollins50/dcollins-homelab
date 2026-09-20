# Runbook: Manage Users (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Identity / SSO |
| Author | Daniel Collins |

---

## When to use this

Use this for adding, modifying, deactivating, or removing users, and for the related (but distinct) task of putting a user into a group. This covers people; for automation/service accounts, see the dedicated section below, they're a different account type with different behavior.

---

## Account types

- **Internal User** — a real person authenticating directly against Authentik (username/password, MFA, etc.). The normal case for a person.
- **External User** — a person authenticating via a federated source (an external identity provider) rather than a locally-set password.
- **Service Account** — non-interactive, for automation (scripts, CI jobs, an app's API calls). Authenticates via an app password or API token, not a login. Cannot use the Authentik UI, cannot do interactive MFA enrollment, and should never be used to represent an actual person.

---

## Steps

### Creating a user (Internal)

1. **Directory → Users.**
2. Select the folder to create the user in (if using folders to organize users).
3. **New User → Internal User.**
4. Fill in required fields:
   - **Username:** must be unique across all user folders
   - **Name:** display name
   - **Email:** used for notifications and email-based flows if configured
5. Create.
6. **Add the user to appropriate group(s)/role(s) now**, a user has no meaningful permissions on their own, see [Manage Permissions](authentik-manage-permissions.md). This is a separate step from creating the account, don't skip it and assume defaults are sufficient.

### Creating a service account

1. **Directory → Users → New User → Service Account.**
2. Set the username and other required fields.
3. On creation, Authentik generates an app password (shown once, store it immediately) or you generate an API token for it afterward, depending on how it'll authenticate.
4. Add the service account to whatever group/role scopes the specific automation permissions it needs, the same least-privilege principle applies to service accounts as to people, don't grant broad permissions out of convenience.
5. Restrict which applications it can reach via bindings/policies if it shouldn't have blanket access.

### Modifying a user

1. Directory → Users, click the edit icon beside the user, or open the user and click Edit from their detail page.
2. Update fields as needed, then save.

### Adding a user to a group

1. Directory → Users, click the user's name to open their detail page.
2. **Groups tab → Add to existing group** (or create a new group first if it doesn't exist yet).

### Deactivating vs. deleting a user

- **Deactivate** when access should be revoked but the account/history should be preserved (e.g. temporary suspension). The account stops being usable but isn't destroyed.
- **Delete** when the account should be permanently removed. This is not generally reversible, confirm this is actually the intent before doing it, deactivation is the safer default for anything ambiguous.

### Cleaning up a lost MFA device

If a user (including yourself) loses access to their authenticator, their enrolled MFA device(s) can be removed from **Directory → Users → [user] → Session/device details**, forcing re-enrollment on next login. Confirm the user's identity through some other means first, removing MFA is effectively a recovery action and should be treated with the same care as a password reset.

---

## Common mistakes

- **Creating a user without adding them to any group**, leaving them with no meaningful permissions and no application access until someone remembers to fix it.
- **Using a Service Account to represent an actual person** (or vice versa) because it was convenient at the time, this breaks the account-type assumptions Authentik makes (e.g. service accounts can't do interactive MFA, can't log into the UI).
- **Deleting instead of deactivating** when the actual need was just to revoke access temporarily.

---

## Related Documentation

- [Manage Permissions](authentik-manage-permissions.md)
- [Add an Application](authentik-add-application.md)

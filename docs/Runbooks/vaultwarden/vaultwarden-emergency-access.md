---
name: vaultwarden-emergency-access
type: runbook
tool: vaultwarden
---

Set up a trusted contact who can request access to the vault if something happens to you. Not configured. Whether Vaultwarden on [[services-host]] has emergency access enabled, and whether anyone is designated, is <FILL IN>.

## What the official Bitwarden page says
- The trusted contact must have an account on the same server. Here that means a Vaultwarden account.
- Two access levels: View (read access to all items in your individual vault, including passwords and attachments) and Takeover (the contact sets a new master password, which replaces yours and removes your two step login methods).
- The contact submits a request. You can approve it at any time, otherwise it is bound by the wait time you chose in advance.
- The page did not list the wait time values or the exact designation steps. Those are in Bitwarden's "Add & Manage Trusted Emergency Contacts" article. Read it before using this.

## Steps (outline)
1. The contact creates a Vaultwarden account, through an admin invitation if signups are off, see [[vaultwarden-disable-signups-and-invite-users]].
2. In your web vault settings, find Emergency access, invite the contact, choose View or Takeover and the wait time.
3. The contact accepts. You confirm them.
4. Test the flow once with a consenting contact and a short wait time, then set the real wait time.

## Decisions to make
- Who. It should be someone you trust fully, since View exposes every password.
- View or Takeover. Takeover wipes your two step login, which is the point in a real emergency and a danger if abused.
- Whether email must work for notices. See [[vaultwarden-configure-smtp]].

## Related
- [[vaultwarden-recover-access]]
- [[vaultwarden]]

Source: https://bitwarden.com/help/emergency-access/

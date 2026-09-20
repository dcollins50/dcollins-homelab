# Runbook: Manage Blocklists (Pi-hole)

| Field | Value |
| --- | --- |
| Applies to | Pi-hole |
| Category | DNS Filtering |
| Author | Daniel Collins |

---

## When to use this

Use this when adding/removing a blocklist source, or when whitelisting a domain that's being incorrectly blocked (including the interaction with Local DNS Records covered in [Add a Local DNS Record](add-local-dns-record.md), check that runbook first if the symptom is "a domain I control isn't resolving").

---

## Steps

### Adding a blocklist source

1. **Pi-hole admin → Adlists** (previously "Group Management → Adlists" in some versions).
2. **Add** a new adlist by URL, paste the raw list URL (most public blocklists are plain-text files hosted on GitHub or similar).
3. Add a comment describing what the list covers, future-you (or anyone else reading this list) shouldn't have to guess what a URL-only entry is for.
4. **Update Gravity** (Tools → Update Gravity, or it may prompt automatically) to actually pull in the new list's domains, adding the adlist entry alone doesn't apply it until Gravity runs.
5. Spot-check that it's actually blocking something expected, don't just assume the update succeeded.

### Removing a blocklist source

1. Adlists, find the entry, disable or delete it.
2. **Update Gravity** again, removal doesn't take effect until Gravity re-runs either.

### Whitelisting a domain

Use when a legitimate domain is being blocked incorrectly, including a domain you control that a blocklist happens to catch (see the Local DNS Record interaction below).

1. **Domains → Whitelist** (Exact or Regex, depending on whether one specific hostname or a pattern needs to be allowed).
2. Add the domain.
3. Test resolution immediately (`dig @<pihole-ip> <domain>`), confirm it now resolves instead of returning a blocked response.

### Blacklisting a specific domain

For blocking something a blocklist source doesn't already cover, without adding an entire new adlist for one domain.

1. **Domains → Blacklist**, add the domain (Exact or Regex).
2. No Gravity update needed for a manually added blacklist entry, it's applied immediately.

---

## Troubleshooting

**A domain that should resolve is being blocked:** Check whether it's actually on an adlist versus the manual blacklist, the fix differs (whitelist entry vs. removing/editing the blacklist entry). Check **Query Log** to see which list matched.

**A domain that should be blocked is resolving anyway:** Check whether it's on the whitelist (intentionally or by mistake), whitelist entries override blocklist matches, an old whitelist entry that's no longer needed is an easy thing to forget about.

**Local DNS Record not resolving despite existing:** This is actually a blocklist issue, not a records issue, see the troubleshooting section in [Add a Local DNS Record](add-local-dns-record.md), a blocklist match takes precedence over a Local DNS Record for the same hostname.

---

## Common mistakes

- **Adding an adlist and forgetting to update Gravity**, the source exists in the list but isn't actually being enforced yet.
- **Adding overly broad blocklists without checking what they actually cover first**, can end up blocking something needed without an obvious explanation why, know what's in a list before adding it, not just that it's popular.
- **Leaving stale whitelist/blacklist entries** for domains that no longer need the exception, periodically worth a review rather than only ever adding.

---

## Related Documentation

- [Add a Local DNS Record](add-local-dns-record.md)

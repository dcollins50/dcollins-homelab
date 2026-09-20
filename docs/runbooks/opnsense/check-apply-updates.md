# Runbook: Check and Apply Updates (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense |
| Category | Maintenance |
| Author | Daniel Collins |

---

## When to use this

This is a recurring maintenance task, not a one-off. The actual cadence practiced in this environment: check roughly every two weeks, but treat "a new version exists" and "it's safe to install now" as two separate questions. A release is deliberately held back if it carries a known, still-unpatched vulnerability, and only applied once a fix for that issue has shipped. This is a considered policy, not just procrastination, so don't skip the review step and update reflexively just because a new version is available.

---

## Steps

### Checking for updates

1. **System → Firmware → Status.**
2. Review what version is currently installed versus what's available.
3. **Before updating, check for known issues with the new version:**
   - Review the OPNSense changelog/release notes for the new version
   - Check whether any security advisories or community reports flag an unpatched vulnerability in that specific release
   - If something's flagged and unresolved, hold off, note it, and re-check on the next cycle rather than updating anyway

### Applying an update

Once a release has been reviewed and isn't being deliberately held back:

1. **System → Firmware → Updates.**
2. Review the changelog for what's actually changing, not just that an update exists, some changes affect config syntax or default behavior and are worth knowing about before they surprise you.
3. **Confirm a recent configuration backup exists** before proceeding (System → Configuration → Backups → Download configuration), so there's a known-good rollback point if the update goes wrong.
4. Start the update and let it complete, don't interrupt or reboot manually mid-process.
5. **After the update, verify core functionality**, not just that the UI loads:
   - WAN connectivity is still up
   - VLAN interfaces are still up with correct IPs
   - Firewall rules are still in place and behaving as expected
   - Suricata is still running and logging
6. **Check version again** (System → Firmware → Status) to confirm it landed correctly.

---

## Common mistakes

- **Updating on a fixed schedule regardless of known issues.** The two-week check-in is for reviewing, not for blind auto-updating, a release with an open CVE stays on hold until it's patched, even if that's several cycles.
- **Skipping the pre-update config backup** because "it's just a routine update," the update that goes wrong is never the one you expected to.
- **Not reading the changelog**, missing a breaking config change that only becomes obvious after something stops working post-update.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Infrastructure](../../infrastructure.md)

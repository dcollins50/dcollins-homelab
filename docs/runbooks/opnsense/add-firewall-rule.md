# Runbook: Add a Firewall Rule (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense |
| Category | Firewall |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a VLAN or host needs new permitted traffic that isn't already covered by an existing rule or alias. Under the default-deny policy, anything not explicitly allowed is blocked, so a new service, a new inter-VLAN path, or a new management access path all start here.

---

## Before you start

- Know the exact source, destination, protocol, and port(s) the traffic needs. Don't default to "any" if the actual requirement is narrower, default-deny only works if rules stay scoped.
- Check whether an alias already exists for the source/destination/port you need (Firewall → Aliases). Reuse an existing alias over hardcoding an IP or port whenever one fits, this environment leans on aliases specifically to keep rules auditable and centrally updatable, see [network.md](../../network.md) for the reasoning.
- Know which interface tab the rule belongs on. OPNSense rules are evaluated per-interface, on a first-match basis, so the rule has to go on the tab matching the traffic's source VLAN, not its destination.

---

## Steps

1. **Navigate to the right interface's rule tab.**
   Firewall → Rules → [Interface] (e.g. LAN, or the specific VLAN tab the traffic originates from).

2. **Click the `+` (Add) button** to create a new rule. Decide whether it needs to go above or below existing rules, since the first matching rule wins and nothing below it is evaluated. Block rules that need to take precedence go above any broader allow rule they'd otherwise be overridden by.

3. **Set the action.**
   - **Pass** to allow the traffic
   - **Block** to silently drop it
   - **Reject** to actively refuse it (rarely needed here, this environment defaults to Block)

4. **Set protocol.** TCP, UDP, TCP/UDP, ICMP, or any, matching the actual traffic type.

5. **Set source.** Use an alias if one exists for this source (a VLAN net, a host group, etc.); otherwise the VLAN's own "net" or a specific host/network.

6. **Set destination.** Same logic, prefer an existing alias (a host alias, a port group) over a raw IP or port list.

7. **Set destination port** (if TCP/UDP). Use a port alias if the destination already has one (e.g. a standard port group), otherwise the specific port(s).

8. **Write a clear description.** Every rule in this environment has a plain-language description, this is what makes rule order and intent readable later. State what the rule allows and why in a few words (e.g. "Allow SOC net to wazuh-manager, agent enrollment").

9. **Save**, then **Apply Changes** on the rules page. Changes don't take effect until applied.

10. **Test the actual traffic path**, don't just trust that the rule looks correct. Confirm from the source host that the intended traffic now works, and confirm nothing unintended also started passing (a rule that's too broad is as much a problem as one that's missing).

---

## Common mistakes

- **Wrong interface tab.** A rule placed on the destination's tab instead of the source's tab silently never matches.
- **Rule order.** A broad "allow any" rule above a more specific block rule will swallow traffic the block rule was meant to catch. New rules should be placed deliberately, not just appended to the bottom.
- **Forgetting to Apply Changes.** Saving a rule doesn't activate it, review-mode changes have to be explicitly applied.
- **Hardcoding a value that already has an alias.** Defeats the point of the alias system, an environment with heavy alias use loses that benefit if some rules bypass it.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [SOP: VLAN Implementation](../../sop/sop-vlan-implementation.md)

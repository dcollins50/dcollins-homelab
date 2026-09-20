# Runbook: Create and Manage Aliases (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense |
| Category | Firewall |
| Author | Daniel Collins |

---

## When to use this

Use this whenever a firewall rule would otherwise need a raw IP, subnet, or port hardcoded into it, and that value is either likely to change or likely to be reused across more than one rule. Aliases are the mechanism this environment relies on to keep the ruleset auditable and centrally updatable rather than scattering the same IP or port across a dozen individual rules, see [network.md](../../network.md) for the full reasoning.

As a rule of thumb: if you're about to type the same IP, host, or port into a second rule, it should already be (or become) an alias.

---

## Before you start

- Check Firewall → Aliases first. An alias for what you need may already exist under a name that isn't obvious, search before creating a duplicate.
- Decide the right alias **type** for what you're representing:
  - **Host(s)** — one or more specific IPs (a single VM, a small fixed group of hosts)
  - **Network(s)** — a subnet (a whole VLAN's net, RFC1918 ranges)
  - **Port(s)** — one or more ports, useful for grouping related ports a service uses
  - **Network group** — a group of other aliases/networks combined (e.g. all Wazuh agent subnets across VLANs)
  - **External (advanced)** — for things like maintained threat-intel lists (bogons, etc.), not typically something you create manually
- Decide the name **before** creating it. Alias names show up in every rule that references them, so a clear, consistent naming convention matters more here than almost anywhere else in the firewall config. This environment's existing pattern is descriptive PascalCase or snake_case tied to the resource's role (e.g. a manager host, a log-ports group, a per-VLAN agent group), not a generic label.

---

## Steps

### Creating a new alias

1. **Navigate to** Firewall → Aliases → Aliases tab.
2. **Click `+` (Add).**
3. **Set the name.** No spaces; keep it descriptive and consistent with existing naming.
4. **Set the type** (Host(s), Network(s), Port(s), Network group, etc.) per the guidance above.
5. **Add content.** Depending on type, this is one or more IPs, CIDR ranges, ports, or other alias names (for a Network group).
6. **Write a description.** Same principle as firewall rules, the description is what makes the alias list readable later without having to guess what an alias is for from its name alone.
7. **Save**, then **Apply Changes**.
8. **Reference the alias** in whatever rule(s) prompted creating it, rather than leaving it unused.

### Editing an existing alias

1. Firewall → Aliases → Aliases, find the alias, click the edit (pencil) icon.
2. Update content as needed. Because rules reference the alias by name rather than by value, editing the alias's content updates every rule that uses it, no need to touch the rules themselves.
3. Save and Apply Changes.
4. **Verify downstream effect.** Since one alias can feed many rules, confirm the change had the intended effect everywhere it's used, not just on the one rule you had in mind when you made the edit.

### Deleting an alias

1. **Check what references it first.** OPNSense will generally block deletion of an alias still in use by a rule, but confirm nothing depends on it before removing it, especially for a Network group alias that might be nested inside other aliases.
2. Remove or update any rules/aliases still referencing it.
3. Delete the alias, Save, Apply Changes.

---

## Common mistakes

- **Creating a near-duplicate alias** instead of finding and reusing the existing one, this is how alias sprawl happens and defeats the point of centralizing values.
- **Vague names** (e.g. a name that doesn't indicate the resource or its role) that force you to open the alias just to remember what it's for.
- **Forgetting to actually use the alias** in the rule that prompted creating it, leaving a hardcoded value in place alongside an unused alias.
- **Not checking Network group nesting** before deleting or heavily editing an alias, since a change can ripple into every group that includes it.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Add a Firewall Rule](add-firewall-rule.md)

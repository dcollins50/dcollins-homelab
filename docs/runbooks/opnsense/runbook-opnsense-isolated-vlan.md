# Runbook: Build a New Isolated VLAN Segment (OPNSense + Managed Switch)

**Category:** OPNSense
**When to use:** Standing up a new network segment for a host that needs strict isolation from other VLANs — e.g. a bastion host, a new trust-boundary service — where "strictly what this host needs, nothing more" is the actual goal, not just organizational tidiness.

## Steps

1. **Switch config**: add the new VLAN ID as *tagged* on the trunk ports connecting the Proxmox host and OPNSense (not access/untagged — those are for single-VLAN end devices only). Both ends of the trunk need to agree on the tagged VLAN list or traffic drops silently.

2. **OPNSense — create the VLAN device**: Interfaces → Devices → VLAN → Add. Parent interface = your existing trunk parent (e.g. `re0`), VLAN tag = the new ID, description = the segment's name.

3. **OPNSense — assign the interface**: Interfaces → Assignments → select the new VLAN device from the dropdown → Add. Then configure it: enable, static IPv4 (this segment's gateway address, `.1` by convention), leave DHCP off if every host on the segment will be statically addressed.

4. **Build aliases before writing rules** — every source host and every external destination should be a named alias, not a raw IP typed into each rule. This is what makes later scoping edits (adding one more allowed destination, tightening a rule) safe and legible instead of a raw-IP hunt.

5. **Write rules scoped to the specific host alias, not the whole VLAN net**, in this order:
   - DNS to the VLAN's own gateway (53, TCP/UDP)
   - NTP to the VLAN's own gateway (123, UDP) — check first whether your firewall already runs its own NTP server (Services → Network Time) before assuming an external time source is needed
   - Whatever specific outbound destinations the host's actual workload requires, each as its own rule with its own alias
   - A default-deny is normal here; nothing needs a catch-all "allow VLAN to internet" rule unless the workload genuinely needs broad outbound access

6. **Package/update access is a common gap** — if the host will run `apt`/`dnf`/etc., the OS's package mirror is very likely CDN-backed (Fastly, Cloudflare, Akamai) with a rotating pool of edge IPs, not a single fixed address. A Host(s) alias with the mirror's hostname will work initially and then start failing sporadically as the CDN routes to different edges. Use the CDN provider's officially published IP range (Networks-type alias) instead, resolved once at write time from the provider's own published list (e.g. `api.fastly.com/public-ip-list`, `cloudflare.com/ips-v4`).

7. **Order matters only when rules can overlap.** With distinct, non-overlapping destination/port combinations per rule (the common case for a narrowly-scoped host), rule order is irrelevant. It only matters when a broader rule could shadow a narrower one below it — check for that specifically before assuming reordering is needed.

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| `apt update` hangs specifically on IPv6 addresses | The mirror's DNS record includes an AAAA entry with no route on this VLAN | Disable IPv6 on the host (`/etc/sysctl.d/`) rather than assuming the firewall rule is wrong |
| `apt update` fails outright on `http://` sources even with a 443-only rule in place | Debian's default sources.list uses `http://`, which the firewall correctly blocks (only 443 was allowed) | Rewrite sources.list to `https://` rather than opening port 80 |
| DNS works for one destination but times out for another, despite an apparently-matching alias | The alias was built from one-time-resolved IPs behind a CDN, not the provider's published range | Step 6 above |
| A brand-new Debian 13 LXC has `systemd-sysctl.service` (and often other core services) failing with `243/CREDENTIALS` | Known Debian 13 + unprivileged LXC incompatibility with systemd's credentials-loading mechanism, unrelated to any custom config | Enable nesting on the container (`pct set <vmid> --features nesting=1`), restart |

---
name: dhcp-dns-rollout
type: project
status: planned
---

Plan for real DHCP address assignment and a consolidated DNS setup across the stack. Planned Sep 23, 2026. Not started. Anything marked "proposed" is not agreed.

A GitHub-ready version of this plan is published at `docs/change-management/dhcp-dns-rollout.md` in the repo.

## Goal
Daniel wants proper DHCP across the stack and expects it to make remote provisioning more reliable. Today everything is static. DHCP was deliberately deferred during the VLAN51 build as a large project for later ([[VLAN51]]). DNS was added to this project the same day: Daniel wants to move off Pi-hole as the main resolver to a more mature setup, with Unbound.

## Decisions (Daniel, Sep 23, 2026)
- No DHCP service is running on OPNsense today.
- DNS migration is part of this project.
- Unbound on [[opnsense]] becomes the resolver for the stack. Pi-hole on [[heimdall]] keeps running for domain blocking.
- VLAN11 is built with DHCP from the start and is the pilot. See [[aiops-vlan-build]].
- Pi-hole failure mode: fail open, with alerts that it failed.
- Daniel wants `homelab.example` only for internet-facing hosts. Internal names use `.internal`. `.local` is ruled out.

## Verified facts (OPNsense documentation, checked Sep 23, 2026)
- ISC DHCP is end of life and gets no security patches. The docs recommend Kea or Dnsmasq. Do not build on ISC.
- Kea cannot register hostnames dynamically with Unbound. Only static reservations are synchronized, and only when Unbound restarts.
- Kea supports reservations, API-driven configuration, and a separate DDNS component for RFC2136 updates to an authoritative DNS server.
- Dnsmasq combines DNS and DHCP in one service.

## Consideration raised
DHCP does not by itself make provisioning more reliable. Reliability comes from the reservation, the DNS record and the firewall rules all existing before the host boots. Past incidents here were DNS, firewall and drifted-config problems, not addressing problems. The real gains are one central place for addresses, no per-host netplan edits, and a provisioning flow that can be automated.

## Current DNS state (from the vault, may be stale)
- [[VLAN20]] DNS is restricted to Pi-hole (192.168.100.1). [[VLAN10]] is restricted to OPNsense. [[VLAN30]] and [[VLAN51]] use Unbound on their gateway. The network MOC says Pi-hole is primary for all VLANs, so these disagree.
- Pi-hole holds local records, most pointing at the internal NPM (10.0.20.30). Unbound holds host overrides for `homelab.example` names. The Sept 15 incident found a stray `::1` AAAA record in Pi-hole answering alongside a correct Unbound A record.
- The Proxmox nodes resolve through Pi-hole.

## Proposed DNS design
- Unbound is the only resolver hosts and DHCP clients see, on each VLAN's gateway address.
- Internal names live only in Unbound. Migrate the Pi-hole local records into Unbound overrides, then delete them from Pi-hole so nothing answers twice.
- Everything that is not an internal name is forwarded to Pi-hole for blocking, and Pi-hole forwards upstream.
- Firewall: each VLAN's DNS rule allows DNS only to that VLAN's gateway. The VLAN20 rule currently allows only Pi-hole and must change. Static hosts, including the Proxmox nodes, get repointed to Unbound. DHCP clients get it automatically.
- Consequence: Pi-hole will see only OPNsense as the client. Per-client stats and group rules there lose meaning. Per-client visibility moves to Unbound query logging, shipped to Elastic if wanted.
- Note: OPNsense's Unbound also has its own blocklist feature (DNSBL). It is an alternative to Pi-hole blocking, not part of this plan.

## Open DNS decisions and proposals
- **Internal domain (decided: `.internal`).** `.local` was ruled out. It is the multicast DNS namespace (RFC 6762). Software is expected to resolve it over multicast instead of forwarding it to a DNS server, so unicast `.local` only works on clients configured to allow it. That is manageable on servers Daniel controls and not realistic for phones, laptops, guests or client environments. Options: `.internal` (reserved permanently by ICANN in July 2024 for private use, resolved by normal DNS, not yet an IETF standard) or `home.arpa` (RFC 8375, longer). Daniel chose `.internal` on Sep 23, 2026. Switching means updating overrides, reissuing certificates that name `homelab.local` hosts, and updating configs that reference those names. Transition path: Unbound serves both zones until `.local` is retired. DHCP hands out the search domain, so decide before building DHCP. Authentik is internal-only but uses an `authentik.homelab.example` name whose OIDC URLs are baked into step-ca and Cloudflare configuration, so it is a candidate exception.
- **Zone apex (proposed, not agreed).** Use `homelab.internal`, so existing names map one to one (`x.homelab.local` becomes `x.homelab.internal`) and other environments, such as a client site, can get their own zone later. DHCP hands out this domain as the search domain.
- **Fail open with alerts (decided).** Proposed implementation: a single forward zone to Pi-hole with Unbound's `forward-first` enabled, so a failed forwarder falls back to normal recursion. Verified caveats: do not add a public resolver as a second forwarder, because Unbound picks among forwarders at random within an RTT band and would bypass Pi-hole while it is healthy. `forward-first` only triggers on SERVFAIL, and fallback can be slow, so test it. It may need a custom Unbound config include on OPNsense. Alerts, through Uptime Kuma to ntfy: Pi-hole unreachable, and a canary check that a domain Pi-hole always blocks does not resolve normally through Unbound, which catches silent bypass. Exact Kuma capabilities to verify in the build.

## Proposed DHCP design
**Three tiers**
1. Static forever: [[opnsense]], [[switch-tl-sg108e]], the four Proxmox nodes, [[heimdall]], and anything that itself provides DNS or DHCP. The Proxmox nodes stay static because corosync ring addresses live in `corosync.conf` (see [[incident-2026-02-23-corosync-stale-ips|Feb 23 corosync incident]]).
2. Reservation by MAC: servers, VMs and LXCs. Same address every time, so firewall aliases and certificates keep working.
3. Dynamic pool: lab VLAN hosts, workstations, phones, laptops.

**Address plan per /24 (proposed):** .1 gateway, .2 to .9 network gear, .10 to .99 reservations, .100 to .199 static or spare, .200 to .250 dynamic pool where needed. Existing hosts already fit (for example [[authentik]] at .10, [[pve-int-stepca]] at .20, [[services-host]] at .30, [[npm-dmz]] at .69).

**Server:** Kea, reservation-first. Reasons: not end of life, API-driven for later automation, reservations sync to Unbound. Alternative: Dnsmasq, which would overlap with Unbound.

**Reservation-only VLANs:** [[VLAN30]], [[VLAN51]] and [[VLAN11]] should have no dynamic pool. Verify that Kea accepts a subnet with reservations and no pool. If it requires a pool, use a small one and record it.

## Migration method
Reserve what a host already has. Create a reservation with its current MAC and current IP, confirm it, then switch the host from static to DHCP at a quiet moment. The IP never changes, so aliases, rules and certificates are untouched. Keep the static config documented until the host has survived a reboot on DHCP.

## Phases
0. **Decisions.** The two open DNS decisions above. Lease times.
1. **DNS consolidation.** Migrate internal names into Unbound, repoint VLANs one at a time, change the DNS firewall rules, remove duplicate records from Pi-hole. Verify each VLAN before the next.
2. **Kea pilot on [[VLAN11]].** Enable Kea on that interface only, with a reservation for the jetson.
3. **Reservation tier, VLAN by VLAN,** in increasing risk: lab, services, SOC, DMZ, trust infrastructure, bastion last.
4. **Remote provisioning.** Proxmox templates with a fixed MAC per clone. Create the reservation and the DNS record together, then boot. Candidate for automation through [[soar-host]].
5. **Hardening and verification.** Long lease times on reservations. Alert on pool exhaustion. Reboot OPNsense once on purpose and confirm hosts keep their addresses. Include DHCP and Unbound configuration in the OPNsense backup routine.

## Risks
- DHCP becomes another dependency at boot. Keeping nodes and network gear static limits this.
- A new host has an address before it has a name, because Kea reservations reach Unbound only on restart.
- DNS consolidation touches every host. Do it VLAN by VLAN.
- A rogue DHCP server on a permissive VLAN could point hosts at its own DNS. Relevant to the lab VLANs.
- Not yet checked: that OPNsense allows DHCP traffic on default-deny interfaces once the server is enabled, and that Unbound is listening on every VLAN interface with an access list for each subnet.

## Related
- [[Network]]
- [[opnsense]]
- [[heimdall]]
- [[aiops-vlan-build]]
- [[opnsense-add-vlan-interface|Add a VLAN Interface]]

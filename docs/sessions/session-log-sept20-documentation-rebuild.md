# Session Log — September 20, 2026 (PM): Documentation Rebuild and Runbook Creation

**Scope:** A full-day documentation session: auditing and rebuilding the entire core doc set against actual current state, reorganizing historical records into a real taxonomy, and authoring the first complete set of operational runbooks for this environment. This session produced no infrastructure changes, it's entirely documentation and process work.

---

## 1. Network Diagram Finalized

Resolved a draw.io export issue (Appearance: Dark export setting vs. the app's own dark-mode UI theme, which don't control the same thing) and added a legend clarifying arrow colors (Cloudflare admin path, Tailscale emergency path, general connectivity, Cloudflare web path). This is the diagram now embedded in the README.

## 2. Core Documentation Rebuilt Against Actual State

`network.md`, `infrastructure.md`, `pki.md`, `soc-stack.md`, `security-lab.md`, and `README.md` had all drifted significantly from reality, wrong node names (`pve-lab`/`pve-SOC` instead of `pve-gateway`/`pve-env2`), VMs that no longer exist (`kalshi-mm`, a phantom VM 700), an incorrect VLAN40/41 split, and a PKI doc that had the Trust Infrastructure VLAN migration backwards. All six were rebuilt section by section against verified current facts, including:

- Physical topology corrected to show Cloudflare Tunnels and Tailscale sitting inline after OPNSense, not branching off before it.
- VLAN10 SOC firewall policy rewritten from the *actual* OPNSense ruleset (pulled directly from the firewall UI), not assumed.
- The Root CA and Intermediate CA are now documented as living on VLAN30 (Trust Infrastructure), reflecting the intended end state, with real IP octets redacted (`10.0.30.x`) since this is public-facing documentation and there's no reason to publish exact addresses for trust infrastructure. **This is ahead of reality**, the actual migration hasn't happened yet, see the open item below.
- Remote Access section rewritten to describe all three parallel paths (Cloudflare Zero Trust, Tailscale, Heimdall WireGuard) per the Sep 18 decision.
- Numbers that couldn't be verified (exact Wazuh agent counts, dashboard counts) were replaced with qualitative descriptions rather than carried forward unverified or guessed at.

## 3. Repo-Wide Audit and Reorganization

Found and fixed a broken link to a `SOP-SEC-001` document that had never actually been committed. Scoped what that hardening SOP should contain, then **deliberately did not publish it**, a list of specific unpatched gaps (Suricata still in IDS-only mode, the CA migration still pending, no SSH bastion yet, no Tailscale key rotation policy, reactive-only OS patching) is more useful to an attacker than to anyone else, so it stays as a private tracker instead of a public repo file.

Reorganized `docs/runbooks/` (which had been a flat dump of postmortems, session logs, and one real runbook all mixed together) into three real categories:

- **`docs/incidents/`** — anything that was a break/fix, renamed with an `incident-review-` prefix
- **`docs/sessions/`** — deliberate work, session logs and migrations
- **`docs/runbooks/`** — genuinely repeatable "how to operate this software" procedures, which was empty until this session's work below

As part of this, `session-log-march30.md` was trimmed to remove content duplicated in `incident-review-march30.md`, and cross-links between the VLAN failure postmortem and its recovery doc were fixed (both had been referencing each other by the wrong filename).

## 4. First Full Set of Operational Runbooks Authored

32 runbooks written across seven tools, each grounded in an actual incident, decision, or standing preference from this environment rather than generic vendor documentation, and cross-verified against each tool's official docs where applicable (Authentik, Elastic ILM, Nginx Proxy Manager, Pi-hole, Proxmox corosync behavior):

| Tool | Count | Notable runbooks |
|---|---|---|
| OPNSense | 7 | Firewall rules, aliases, VLAN interfaces, Suricata, DNS forwarding, update policy, agentless SSH monitoring |
| Proxmox | 3 | VM/LXC creation, cluster quorum troubleshooting (built around the real `/etc/pve/corosync.conf` vs `/etc/corosync/corosync.conf` lesson from the January VLAN failure incident) |
| Authentik | 6 | Application/provider setup, outposts, users, permissions, MFA enforcement, backup/restore |
| ELK | 6 | Log pipelines, TLS, ILM retention (built around the actual shard-limit outage), log-shipping troubleshooting, Kibana Lens conventions, Wazuh dashboard import |
| Nginx Proxy Manager | 3 | Proxy hosts, TLS certs, and a routing runbook built directly around the real Filebeat/port-9200 incident |
| Pi-hole | 3 | Local DNS records, blocklist management, and a DNS-failover contingency runbook left with an open item pending confirmation of the current dual-Pi-hole arrangement |
| Internal PKI | 4 | Leaf cert issuance, revocation, bringing the Root CA online, trust store distribution |

The README was updated with a full Runbooks section, and separately corrected for CompTIA Network+ completion (June 2026, previously shown as in-progress).

---

## Open items carried forward

- **PKI migration to the Trust Infrastructure VLAN** is the next real infrastructure task, the documentation now describes the CAs as already on VLAN30, but the actual migration off the flat 10.0.0.x network hasn't happened yet. This is the explicitly named next open item going into the following session.
- A candidate runbook, `migrate-intermediate-ca-to-stepca.md`, was discussed but deliberately not written yet, since that migration hasn't happened either; better to document what actually happened once it's real than to write a speculative procedure now.
- The Pi-hole dual-DNS arrangement from the Sep 13 restart (a house Pi as primary for the personal workstation, Heimdall's Pi-hole as secondary) was never confirmed as still active, reverted, or partial, `troubleshoot-primary-dns-failover.md` has this flagged as an open item rather than asserting an answer.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Internal PKI](../../pki.md)
- [Infrastructure](../../infrastructure.md)

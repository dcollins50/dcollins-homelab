---
name: Welcome
type: home
status: live
---

Home base for the dcollins-homelab vault. Start here, not in a search bar.

## Read this first
- [[Conventions]] — folder rules, frontmatter schema, naming, how a session should pick up context

## System overviews (start here for "what does X look like")
- [[Infrastructure]] — Proxmox cluster, physical nodes, supporting hardware
- [[Network]] — VLAN design, firewall philosophy, remote access paths
- [[PKI]] — internal CA hierarchy, cert issuance procedure
- [[Security-Lab]] — isolated pentest/malware lab (VLAN40/41)
- [[SOC-Stack]] — ELK, Wazuh, SOAR, IRIS
- [[Tailscale]]: break-glass remote admin path, tailnet policy, enrolled machines

## Browse by kind
- **Hosts/** — one note per physical node, VM, or LXC
- **Network/** — one note per VLAN (VLAN1 through VLAN60)
- **Runbooks/** — single-tool "how do I do X" procedures, organized by tool
- **SOPs/** — multi-tool processes with real dependencies between steps
- **Decisions/** — architectural calls and the reasoning behind them
- **Incidents/** — postmortems: what broke, why, what changed
- **Sessions/** — raw dated session logs
- **Projects/** — in-progress or completed multi-session builds
- **Archive/** — superseded content, kept for reference

## In progress right now
- [[sop-ssh-bastion-build|SSH Bastion Build]] — VLAN51, firewall scoping and the bastion LXC are done. WARP enrollment is blocked by a QUIC error, sshd cert trust and the SSH cutover are not started
- [[sop-stepca-cutover|step-ca Cutover]] — step-ca running the real trust chain, no service switched over yet
- [[sop-soc-alert-pipeline|SOC Alert Pipeline]] — Kuma→Shuffle→IRIS works, Wazuh→Shuffle doesn't exist yet

## Known gaps
See the "Known gaps" section in [[Conventions]] for what's still rough: no delete capability on the filesystem connector, unreviewed legacy content in Incidents/Sessions/Runbooks/SOPs, and three historical incident docs that use pre-rename VLAN addressing.

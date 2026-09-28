---
name: obsidian-vault-reorg-sep23
type: decision
status: live
---

## Decision
Restructured the newly-seeded Obsidian vault from the flat `docs/` layout (copied straight from the GitHub repo) into a linked, atomic structure: Hosts/, Network/, Runbooks, SOPs, Decisions, Projects, plus two additions made without prior sign-off — Incidents and Sessions — and Archive for anything retired.

## What was agreed beforehand
Six folders: Hosts/, Network/, Runbooks/, SOPs/, Decisions/, Projects/. Existing `runbooks/` and `sop/` folders kept and capitalized. Single-topic files (infrastructure.md, network.md, pki.md, security-lab.md, soc-stack.md) broken into atomic per-host/per-VLAN notes.

## Calls made without stopping to confirm
- **Added Incidents/** for the 13 existing postmortem files (docs/incidents/). These didn't fit cleanly into Decisions/ (a decision record isn't the same shape as a postmortem) or Projects/, so it became its own folder rather than forcing a fit.
- **Added Sessions/** for the 12 existing raw session-log files (docs/sessions/). Kept as a chronological log folder distinct from Decisions/, on the reasoning that Decisions/ should hold distilled *why*, not a session-by-session transcript.
- **docs/soc/ (5 phase files + Implementation/ subfolder)** moved into **Projects/soc-stack-buildout/** rather than Runbooks or SOPs, since they read as a project's phased build history rather than repeatable procedures.
- The five original overview docs were rewritten as short MOC (Map of Content) notes at the vault root, keeping conceptual/procedural material and linking out to the new Hosts/Network atomic notes instead of repeating host tables. Originals moved to Archive/docs-original/ rather than deleted (the filesystem connector has no delete capability).

## Open for correction
Any of the above can be renamed, merged, or moved if it doesn't match how this is actually meant to work. Nothing here is final, just a working structure to build on.

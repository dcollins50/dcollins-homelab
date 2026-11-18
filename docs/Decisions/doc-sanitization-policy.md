---
name: doc-sanitization-policy
type: decision
status: live
---

## Decision
Decided Oct 1, 2026. The vault is the only source of truth. The public mirror in the repo's `docs/` folder is generated from it by a script, never edited by hand, and the script never writes to the vault.

- **Run on demand or automatically.** The sanitizer can still be run by hand ([[sanitize-publish-doc]]). Since Oct 2, 2026 it also runs automatically on every note create and save, see Automation below.
- **Mask, do not strip.** The mirror keeps the network structure (VLANs, subnets, which host does what) but hides anything that identifies a specific person, account or public endpoint.
- **One-way.** The vault is opened read-only. The script refuses to write to any path inside the vault.
- **Rules stay local.** The real values to hide live in `Scripts/sanitize-rules.json`. `Scripts/` is gitignored, so those values never reach GitHub.

## What gets masked
- Internal IPs: last octet replaced with `x` (`10.0.10.x` becomes `10.0.10.x`). Subnet notation like `10.0.10.0/24` is left alone so VLAN layout stays readable. Setting `ip_mode` in the rules file to `none` or `all` changes this.
- Tailscale addresses become `100.x.x.x`. Tailnet names become `[tailnet].ts.net`.
- Any public IP not on the allow list (public DNS resolvers, Cloudflare ranges) becomes `[public-ip]`.
- Emails, MAC addresses, UUIDs, Windows user paths, SSH public keys, private key blocks and recognizable API tokens.
- Literal values from the rules file: the VPS address and provider hostname, the real domain, usernames, the collaborator's name and Tailscale account, the workstation hostname.

## What it does not do
- It does not judge meaning. A note that explains an unpublished weakness stays unpublished only if Daniel does not run the script on it.
- It does not sanitize images such as `network-diagram.png`.
- Names of people who are not on the rules list pass through. "Daniel" is not masked.
- Heuristic findings (a long hex string, something that looks like a password assignment) are reported for review, not changed.

## Safety checks built in
- After sanitizing, the script runs its rules over its own output. If anything would still change, the file is not written and the run exits with code 2.
- Public IPs that get masked are listed in the report so a version number mistaken for an IP is easy to spot.

## Ruled out
- Editing notes in place or keeping a "clean" copy inside the vault: two sources of truth.
- A pre-commit hook: it would block commits instead of preparing the mirror ahead of time.
- Auto-commit or auto-push: Daniel reviews with `git diff --staged` and pushes himself.
- Stripping all IPs: loses the structure that makes the repo useful as a portfolio and as a template for the consulting offering.

## Automation (decided Oct 2, 2026)
- **Triggers:** a background watcher that polls the vault, plus an optional Obsidian Shell commands trigger. Both call the same engine, so they cannot disagree. The watcher alone catches notes made in Obsidian and notes Claude writes through the filesystem connector. The Obsidian trigger is belt and braces and adds a third-party plugin with full vault access.
- **Every save, not just creation:** a note is re-sanitized after it has stopped changing for a few seconds, so the mirror never goes stale.
- **Publish by default:** every note is mirrored unless its frontmatter says `publish: false`. Marking a published note private removes its mirror copy and stages the removal. Anything already pushed stays in git history.
- **Held, not published:** a note with review items (possible secret, long hex string) is not written or staged, and a Windows notification is shown. A known-safe string can be added to `review_ignore` in the rules file.
- **Git step:** `git add` on changed `docs/` paths only. No commit, no push. A repo in the middle of a merge or rebase is left alone.
- **Orphans:** notes deleted or renamed in the vault leave their mirror copy behind and are listed as orphans by `--status`, never auto-deleted.
- Setup and day-to-day use: [[sanitize-automate-publish]].

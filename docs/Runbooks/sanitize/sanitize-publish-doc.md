---
name: sanitize-publish-doc
type: runbook
tool: sanitize
---

Produces a masked copy of vault notes in the repo's `docs/` folder. Policy and reasoning: [[doc-sanitization-policy]]. Needs Python 3.8 or newer (in WSL: `python3`).

## Run it
From the repo root (`C:\Users\[user]\dcollins-homelab`, or `~/` equivalent under `/mnt/c/Users/[user]/dcollins-homelab` in WSL). Paths are relative to the vault.

```
python3 Scripts/python/sanitize-doc.py Hosts/nimbus.md
python3 Scripts/python/sanitize-doc.py Hosts/nimbus.md Decisions/dns-architecture.md
python3 Scripts/python/sanitize-doc.py Runbooks/wazuh
python3 Scripts/python/sanitize-doc.py --all
```

- `--dry-run` shows what would change and writes nothing. Use it first on anything new.
- `--stdout` prints one sanitized note to the terminal.
- `--out <folder>` writes somewhere other than `docs/`. A folder inside the vault is refused.

Output keeps the same relative path, so `Hosts/nimbus.md` lands at `docs/Hosts/nimbus.md`.

## Read the report
Each file prints a change count by rule. Then check:
- **REVIEW lines**: the script found something that might be a secret or a key hash and did not change it. Open the note and decide.
- **public IPs masked**: confirm none of them is a version number.
- **Exit code**: 0 clean, 1 written but needs review, 2 a file failed its leak check and was not written, 3 safety or usage error.

## Publish
The script only writes files. Committing and pushing is a separate step, and `git diff docs/` is the last look before it.

## Change what gets masked
Edit `Scripts/sanitize-rules.json` (local, gitignored). Add a literal to hide a new name or hostname, add to `keep_ip_allow` for a public IP that is fine to show, or change `ip_mode`. Re-run with `--dry-run` to see the effect.

## Known limits
- Files already in `docs/` that no longer exist in the vault (for example `docs/Hosts/vps-redirector.md`, left from the Sep 28 mirror) are never deleted by the script. Remove them by hand in git.
- Wikilinks are kept as written. A mirrored note can link to a note that was never mirrored.

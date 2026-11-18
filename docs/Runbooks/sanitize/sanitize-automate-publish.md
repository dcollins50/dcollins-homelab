---
name: sanitize-automate-publish
type: runbook
tool: sanitize
---

Sets up and operates the automatic mirror. Policy and reasoning: [[doc-sanitization-policy]]. Manual runs: [[sanitize-publish-doc]].

## What runs
`Scripts/python/publish-sync.py` is the engine. It reads the vault, sanitizes with `sanitize-doc.py`, writes to `docs/`, and runs `git add` on the changed `docs/` paths. It never writes to the vault and never commits or pushes.

| Mode | Used by | What it does |
|---|---|---|
| `--watch` | Windows scheduled task | Polls the vault every 5 seconds, processes a note once it has been unchanged for 8 seconds, catches up on everything at startup |
| `--files <note>` | Obsidian trigger | One-shot for the note that just changed |
| `--reconcile` | You, any time | Sweeps every note, fixes anything out of step |
| `--status` | You, any time | Shows held notes, leak-check failures and orphans |

A lock file keeps the watcher and the Obsidian trigger from running at the same time. Log: `Scripts/logs/publish-sync.log`.

## Install the watcher (once)
From the repo root in PowerShell:
```
powershell -ExecutionPolicy Bypass -File Scripts\PowerShell\install-publish-watcher.ps1
```
It checks that WSL has python3, registers a scheduled task named `homelab-publish-watcher` that starts at logon and restarts if it dies, and starts it. Remove it with `Unregister-ScheduledTask -TaskName homelab-publish-watcher -Confirm:$false`.

## Optional: Obsidian trigger
1. Settings, Community plugins, install **Shell commands** (by Taitava).
2. Add a shell command: `wsl.exe -e python3 /mnt/c/Users/[user]/dcollins-homelab/Scripts/python/publish-sync.py --files "{{event_file_path:relative}}"`
3. On that command, open Events and enable **File created** and **File content modified**. Set the execution mode to **Execute after cooldown** with about 10 seconds, because Obsidian saves often.
4. The plugin runs with full vault access and is not sandboxed. The watcher already covers every change, so this layer is redundant by design.

## Day to day
1. Write or edit notes as normal. Within about 15 seconds the sanitized copy appears in `docs/` and is staged.
2. When you want to publish: `git diff --staged`, `git commit`, `git push`.
3. A Windows notification means something needs you. Run `wsl.exe -e python3 Scripts/python/publish-sync.py --status`.

## Keep a note out of the mirror
Add `publish: false` to its frontmatter. If it was already mirrored, the copy is removed and the removal staged on the next run. It stays in git history if it was pushed before.

## When a note is held
The review line names the note and line. Either fix the note, or if the flagged string is public (a key fingerprint, a hash), add that string to `review_ignore` in `Scripts/sanitize-rules.json`. Then save the note again or run `--reconcile`.

## Known limits
- The Windows scheduled task and the toast notification could not be tested from the cloud session that built them. First run on the workstation is the real test: check the log after saving a note.
- The watcher runs in WSL, so it only runs while the Windows user is logged in and the task is alive.
- Staging uses `git.exe` from WSL when available so it matches Windows git settings.

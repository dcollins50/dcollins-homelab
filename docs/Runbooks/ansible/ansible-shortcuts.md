---
name: ansible-shortcuts
type: runbook
tool: ansible
---

Three small commands that replace the repeated steps around every change. They are personal setup on two machines and are not part of the `soc-ansible` repo, so a rebuilt machine needs them added again.

## On pve-ansible: `soc-sync` and `soc-ready`
Both live in `~/.soc-shortcuts`, loaded from `~/.bashrc` with `source ~/.soc-shortcuts`.
- `soc-sync` pulls the repo and verifies the signature of the newest commit, see [[ansible-verify-signed-commit]]. If either step fails it prints "do not run a playbook" and stops.
- `soc-ready` sets `ACHILLES_HOST` (default is the server's current Tailscale address, so give it the new address after a rebuild: `soc-ready <address>`), starts the ssh agent if there is none, loads the Ansible key for an hour, and moves into the repo. It replaces the three checks in [[ansible-run-playbook]] and the agent steps in [[ansible-load-ssh-agent-key]].

## On the workstation: `soc-ship "message"`
A PowerShell function in the user profile (`Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` on Windows PowerShell 5). It stops if any file under a `vault` folder does not start with the vault header, stages changes to files Git already tracks, shows the list, asks y or n, then commits with a signature and pushes. A new file must be added with `git add` first. The signing and push passphrase prompts still appear.

## Tested
`soc-sync` was run on the live control node and its failure path on a scratch repository. `soc-ship` was run for real on 2026-10-25 (list, confirm, signed commit, push). `soc-ready` was run on the control node by hand only through its pieces, not as one command; the vault-header guard in `soc-ship` has not met an unencrypted vault file yet.

Next: [[ansible-run-playbook]].

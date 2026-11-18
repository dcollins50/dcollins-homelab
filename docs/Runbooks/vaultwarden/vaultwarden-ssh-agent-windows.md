---
name: vaultwarden-ssh-agent-windows
type: runbook
tool: vaultwarden
---

Use the Bitwarden desktop app as the SSH agent on the Windows workstation, so keys stored as SSH key items in Vaultwarden can sign SSH logins. Daniel chose passphrase only storage for `homelab/keys` on Oct 3, 2026, so private keys are not in the vault today. This runbook applies only if that decision changes, see [[vaultwarden-store-ssh-key-passphrases]].

## Steps
1. Install the Bitwarden desktop app and log in to the self hosted server.
2. Settings, enable SSH agent. Set the option for asking authorization when the agent is used, per your comfort.
3. On Windows, the agent uses the OpenSSH service slot. Disable the native "OpenSSH Authentication Agent" in Services, startup type Disabled, so the two do not collide.
4. Test:
   ```powershell
   ssh-add -L
   ```
   Listed public keys mean the agent is serving.
5. Add keys as SSH key items in Vaultwarden (generate there or import).
6. Connect to a host and approve the prompt in the desktop app.

## Known limits
- Requires the desktop app to be running and the vault unlocked.
- Commit signing needs Git 2.34 or newer and OpenSSH 8.8 or newer.
- WSL is not covered in the official page. The existing runbook [[ssh-wsl-agent-setup]] covers key use from WSL the older way. Whether WSL can reach the Windows agent was not verified.
- An SSH key item has no passphrase field, and importing a passphrase protected key had a reported bug (bitwarden/clients issue 13878, Vaultwarden discussion 5331, checked Oct 3, 2026). Re-check before relying on it.

## Related
- [[vaultwarden-ssh-agent-linux]]
- [[vaultwarden-store-ssh-key-passphrases]]
- [[ssh-key-rollout-and-verify]]

Source: https://bitwarden.com/help/ssh-agent/

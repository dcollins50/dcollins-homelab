---
name: ssh-wsl-agent-setup
type: runbook
tool: ssh
---

Get SSH key auth working from a WSL terminal on the Windows workstation, when a key already exists under the Windows user profile but WSL can't see or use it yet. Written up from the Sep 25, 2026 SSH key rollout session.

## The problem

WSL has its own Linux home directory (`/home/<wsl-user>`), completely separate from the Windows user profile. A key that already works for SSH from PowerShell/Windows tools doesn't automatically exist in WSL's `~/.ssh`, even though WSL can see the Windows filesystem through `/mnt/c/`.

## Procedure

1. **Check whether WSL already has its own key** before assuming one needs to be copied or generated:
   ```bash
   ls -la ~/.ssh
   ```

2. **If empty, check the Windows profile for an existing key** (adjust the username):
   ```bash
   ls -la /mnt/c/Users/<username>/.ssh
   ```

3. **Copy it into WSL's own `~/.ssh`** rather than referencing the `/mnt/c/` path directly every time (cleaner for anything else in WSL that expects a normal `~/.ssh`):
   ```bash
   mkdir -p ~/.ssh
   cp /mnt/c/Users/<username>/.ssh/id_ed25519 ~/.ssh/
   cp /mnt/c/Users/<username>/.ssh/id_ed25519.pub ~/.ssh/
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/id_ed25519
   chmod 644 ~/.ssh/id_ed25519.pub
   ```
   The `chmod` steps matter: everything under `/mnt/c/` mounts as world-writable (`rwxrwxrwx`) by default under WSL, and SSH refuses to use a private key with permissions that open. Copying to WSL's native filesystem and re-setting permissions fixes this.

4. **Start the agent and load the key** (this does not persist across terminal sessions, see Notes):
   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```
   If the key is passphrase-protected, `ssh-add` prompts for it here, enter it once per agent session.

5. **Confirm the key is actually loaded**:
   ```bash
   ssh-add -l
   ```
   Should show the key's fingerprint. "Could not open a connection to your authentication agent" at this step means step 4's `eval` either wasn't run, or was run in a different terminal window, agent state is per-session, not global.

## Notes

- **`ssh-agent` does not persist across WSL terminal restarts.** Closing the terminal and opening a new one means redoing steps 4-5 (agent start + key add + passphrase entry) before key auth works again in that new session. There's no step missed here, this is just how it works; plan for it rather than being surprised by it each time.
- Don't run `ssh-add -l` (or most of these commands) with `sudo`, agent state belongs to the normal user session, not root. A sudo password prompt at this step is a sign something's being run with unneeded elevation, not a real blocker.

## Related
- [[ssh-key-rollout-and-verify]]

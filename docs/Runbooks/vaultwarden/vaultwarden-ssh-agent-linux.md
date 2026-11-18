---
name: vaultwarden-ssh-agent-linux
type: runbook
tool: vaultwarden
---

Point SSH on a Linux desktop at the Bitwarden desktop SSH agent. Only relevant if the Bitwarden desktop app runs on that Linux machine. Servers like [[services-host]] do not run it.

## Steps
1. Enable the SSH agent in the desktop app settings, as in [[vaultwarden-ssh-agent-windows]].
2. Find the socket. Per the official page:
   - Standard install: `/home/<user>/.bitwarden-ssh-agent.sock`
   - Snap: `/home/<user>/snap/bitwarden/current/.bitwarden-ssh-agent.sock`
   - Flatpak: `/home/<user>/.var/app/com.bitwarden.desktop/data/.bitwarden-ssh-agent.sock`
3. Export it in the shell profile so it persists:
   ```bash
   echo 'export SSH_AUTH_SOCK=$HOME/.bitwarden-ssh-agent.sock' >> ~/.bashrc
   ```
   Use the right path for snap or flatpak.
4. Test with `ssh-add -L`.

## Related
- [[vaultwarden-ssh-agent-windows]]
- [[ssh-key-rollout-and-verify]]

Source: https://bitwarden.com/help/ssh-agent/

---
name: vaultwarden-bitwarden-cli-setup
type: runbook
tool: vaultwarden
---

Install the Bitwarden CLI (`bw`) and point it at the self hosted Vaultwarden. The CLI is the foundation for scripts and later Ansible or Semaphore use, see [[vaultwarden-use-cli-in-scripts]].

Not yet installed anywhere in this homelab as of Oct 3, 2026. Where to install it: <FILL IN: workstation, WSL, or an automation host>.

## Steps
1. Install. Either download the native executable for the platform, or with npm:
   ```bash
   npm install -g @bitwarden/cli
   ```

2. Point it at the self hosted server before logging in. This is the step people miss, otherwise it talks to Bitwarden's cloud:
   ```bash
   bw config server https://<vaultwarden address>
   ```
   If the address uses a certificate from the internal CA, the machine must trust that CA. See the PKI runbooks and [[PKI]].

3. Log in interactively. It prompts for email, master password and the two step code:
   ```bash
   bw login
   ```

4. Unlock and export the session key. Unlocking prints a session key; `bw` needs it in `BW_SESSION` for any vault command:
   ```bash
   bw unlock
   export BW_SESSION="<session key printed by bw unlock>"
   ```

5. Test:
   ```bash
   bw sync
   bw list items --search pve-gateway
   ```

6. Lock and log out when done:
   ```bash
   bw lock
   bw logout
   ```

## Cautions
- Never type the master password on the command line in a way that lands in shell history.
- A `BW_SESSION` value is as powerful as the unlocked vault. Do not log it or paste it.

## Related
- [[vaultwarden-use-cli-in-scripts]]
- [[vaultwarden]]

Source: https://bitwarden.com/help/cli/

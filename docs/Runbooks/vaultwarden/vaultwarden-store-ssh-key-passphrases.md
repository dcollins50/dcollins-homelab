---
name: vaultwarden-store-ssh-key-passphrases
type: runbook
tool: vaultwarden
---

Store SSH key passphrases in the `homelab/keys` folder, with the key file staying on disk. This is the approach Daniel chose on Oct 3, 2026. The alternative, storing the whole private key as an SSH key item, was rejected for now.

## What an item holds
- Type: Login.
- Name: the key's file name, for example `racknerd_vps`, `dev_git`, or the lab key.
- Username: leave blank. A key has no username.
- Password: the key's passphrase.
- Notes: where the private key file lives and which hosts trust it. Never the private key itself.
- Folder: `homelab/keys`.

## Steps
1. List the keys on the machine: `ls -l ~/.ssh`.
2. For each key that has a passphrase, create one Login item as above.
3. Check that the item works: load the key with `ssh-add ~/.ssh/<key>` and paste the passphrase from the vault when prompted.
4. Delete throwaway test keys (the `ssh-test-key` files from the Oct 3 test) once nothing depends on them.
5. If a key has no passphrase, it is a risk on its own. Consider adding one with `ssh-keygen -p -f ~/.ssh/<key>`, then store the new passphrase.

## Why not the SSH key item type, for now
- The item has no passphrase field.
- Importing a passphrase protected key had a reported bug (bitwarden/clients issue 13878, Vaultwarden discussion 5331, checked Oct 3, 2026).
- The SSH agent needs the desktop app, and WSL is not documented. See [[vaultwarden-ssh-agent-windows]].

## Related
- [[ssh-key-rollout-and-verify]]
- [[ssh-bastion-certificate-login]]
- [[vaultwarden-add-homelab-login]]

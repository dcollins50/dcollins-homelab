---
name: vaultwarden-export-vault
type: runbook
tool: vaultwarden
---

Export the contents of a Vaultwarden vault as a second kind of backup, or to move to another tool. This does not replace the server backup in [[vaultwarden-backup-data]], which covers every user and attachment on the server.

## Web vault
1. Tools, then Export.
2. Choose the source (My vault), a format, and for an encrypted export the type (account restricted or password protected).
3. Export, then confirm with the master password.

## CLI
```bash
bw export --format json --output <folder>/
```
Default format is `.csv`. See [[vaultwarden-bitwarden-cli-setup]].

## Formats and what is missing
- `.json` and `.csv` are plaintext.
- `.json` (Encrypted) is available, either account restricted or password protected.
- A `.zip` with attachments exists for individual vaults.
- No format includes trash items or Sends.
- Only `.json` includes cards, identities, passkeys and SSH keys.

## Cautions
- A plaintext export is every password in one readable file. Do not email it or leave it in Downloads. Delete it right after use, and empty the recycle bin.
- Prefer the encrypted JSON export. Store the export password outside Vaultwarden.
- Do the export on a trusted machine only.

## Related
- [[vaultwarden-backup-data]]
- [[vaultwarden-import-from-bitwarden-cloud]]

Source: https://bitwarden.com/help/export-your-data/

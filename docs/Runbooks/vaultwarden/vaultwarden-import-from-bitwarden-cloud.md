---
name: vaultwarden-import-from-bitwarden-cloud
type: runbook
tool: vaultwarden
---

Move items from a Bitwarden cloud account into the self hosted Vaultwarden. This was how the vault was first populated, from a conversation on the Bitwarden client and self hosted server (Sep 2026). Reconstructed from that chat, not re-run.

## Steps
1. Export from the Bitwarden cloud vault as `.json`, following [[vaultwarden-export-vault]].
2. In the Vaultwarden web vault, Tools, Import data. Pick the Bitwarden (json) format and choose the file.
3. Review the result before deleting anything in the cloud account.
4. Delete the export file securely when done.

## Caveats
- Importing twice creates duplicate items. If a first import was partial, clear it first or expect to clean up.
- File attachments are not included in a JSON export. Use the zip export with attachments, or move them by hand.
- Trash items and Sends are not exported.
- After import, sort items into the `homelab/` subfolders and rename them per [[vaultwarden-add-homelab-login]].

## Related
- [[vaultwarden-export-vault]]
- [[vaultwarden]]

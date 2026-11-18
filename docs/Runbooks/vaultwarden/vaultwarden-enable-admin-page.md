---
name: vaultwarden-enable-admin-page
type: runbook
tool: vaultwarden
---

Turn on the Vaultwarden admin page with a hashed admin token. State on [[services-host]] is not yet recorded: whether the admin page is on, and whether the token is hashed, is <FILL IN>.

## Steps

1. Make sure Vaultwarden is behind HTTPS first. The official docs strongly recommend this before enabling the admin page, to avoid man in the middle attacks. See [[vaultwarden-npm-proxy-host]].

2. Generate the admin password as a long random string, and store it in Vaultwarden under `homelab/apps` or `homelab/identity` once you have a working login:
   ```bash
   openssl rand -base64 32
   ```

3. Hash it with Argon2id using Vaultwarden's own command. It prompts for the password and prints the hash:
   ```bash
   docker run --rm -it vaultwarden/server /vaultwarden hash
   ```

4. Put the hash in `ADMIN_TOKEN`.
   - In a compose file `environment:` block, every `$` in the hash must be doubled to `$$`. The hash contains five of them.
   - In an `.env` file, keep single `$` and wrap the value in single quotes.

5. Restart the container (`docker compose up -d` or re-create it), then browse to `https://<vaultwarden address>/admin` and log in with the original password, not the hash.

## Notes
- Admin sessions last 20 minutes by default (`ADMIN_SESSION_LIFETIME`).
- To invalidate sessions, the official docs say to remove `rsa_key.pem` and restart. Check [[vaultwarden-backup-data]] first, since that file is also in the data folder.
- Settings saved from the admin page are stored in `config.json` in the data folder (the official backup guide lists it as recommended). Back up that file.
- Never paste the token into chat or a note.

## Related
- [[vaultwarden]]
- [[vaultwarden-harden-container]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page

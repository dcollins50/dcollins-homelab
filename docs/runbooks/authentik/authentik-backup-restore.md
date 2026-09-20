# Runbook: Backup and Restore (Authentik)

| Field | Value |
| --- | --- |
| Applies to | Authentik |
| Category | Maintenance / Disaster Recovery |
| Author | Daniel Collins |

---

## When to use this

Authentik is the SSO/IdP for this environment, losing it without a usable backup means losing authenticated access to everything sitting behind it, not just Authentik itself. Use this runbook before any risky change (version upgrade, config overhaul), and periodically as standing practice, not only when something's already gone wrong.

---

## What actually needs to be backed up

**The PostgreSQL database is the critical piece.** It holds essentially all persistent Authentik data, users, groups, policies, flows, providers, applications, and configuration. Losing it without a backup means full data loss, there's no other source of truth to rebuild from.

Beyond the database, back these up if they're in use in this deployment:
- **Media directory** — application icons, flow backgrounds, uploaded files (only relevant if not using external/S3 storage)
- **Certs directory** — only needed if relying on filesystem-stored TLS certs rather than certs already imported into Authentik itself (imported certs live in the database, not the filesystem)
- **Custom templates directory** — only if the default UI has been customized
- **Compose file / environment config** — the `.env` file and `docker-compose.yml` (or equivalent), needed to redeploy the same configuration, not just restore data into a fresh instance

---

## Steps

### Backing up the database

Exact commands depend on how Authentik is actually deployed here (Docker Compose vs. another method), but the underlying approach is standard PostgreSQL tooling:

```bash
# From the host running the PostgreSQL container/service:
pg_dump -U <authentik-db-user> -d <authentik-db-name> -Fc > authentik-backup-$(date +%Y%m%d).dump
```

Or, if running inside a Docker Compose deployment:

```bash
docker compose exec postgresql pg_dump -U authentik -d authentik -cC > authentik-backup-$(date +%Y%m%d).sql
```

- Exclude the PostgreSQL system databases (`template0`, `template1`), they're not part of what needs restoring.
- **Store the backup somewhere other than the database host itself.** A backup that lives on the same disk as the thing it's protecting against isn't a real backup.
- Confirm the dump file actually exists and looks non-empty/non-trivial before considering the backup complete, don't assume the command succeeded just because it didn't print an error.

### Backing up supporting directories

```bash
tar -czf authentik-support-$(date +%Y%m%d).tar.gz media/ certs/ custom-templates/ .env docker-compose.yml
```

Adjust paths to match the actual deployment layout, only include directories actually in use.

### Restoring

1. **Restore the PostgreSQL database first**, before bringing Authentik back into service, an Authentik instance pointed at an empty or partial database will misbehave rather than gracefully wait.
   ```bash
   pg_restore -U <authentik-db-user> -d <authentik-db-name> --clean --if-exists authentik-backup-<date>.dump
   ```
   (Use `psql` instead of `pg_restore` if the backup was taken as a plain `.sql` dump rather than the custom `-Fc` format.)
2. **Verify the restored database looks complete** before reconnecting Authentik to it, spot-check that expected users/applications are present.
3. Restore the supporting directories (media, certs, templates, compose config) to their expected locations.
4. Bring Authentik itself back up, pointed at the restored database.
5. **Verify end-to-end**, not just that Authentik's UI loads: confirm login works, confirm at least one protected application (e.g. the ntfy forward-auth path) still authenticates correctly.

---

## Common mistakes

- **Backing up only the database and forgetting the compose/env config**, restoring data into a fresh deployment that doesn't match the original configuration (different secret key, different outpost setup, etc.) can cause subtle breakage.
- **Never actually testing a restore.** A backup that's never been restored from is unverified, the failure mode (a corrupt or incomplete dump) is usually invisible until the moment it actually matters.
- **Storing the backup on the same host/disk** as the live database, defeats the purpose if that host is what fails.
- **Treating "authentik does not support downgrading"** as irrelevant to backups, this is exactly why a pre-upgrade backup matters: there's no built-in undo if an upgrade goes wrong.

---

## Related Documentation

- [Internal PKI](../../pki.md)
- [Manage Users](authentik-manage-users.md)

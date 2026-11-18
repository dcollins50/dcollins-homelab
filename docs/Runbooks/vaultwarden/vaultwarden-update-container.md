---
name: vaultwarden-update-container
type: runbook
tool: vaultwarden
---

Update the Vaultwarden container on [[services-host]] to a newer release. Plan written Oct 3, 2026 after finding the server was behind on security fixes. The real current version was not yet confirmed at the time, see [[vaultwarden-find-version-and-deployment]].

## Before you start
1. Take a backup and copy it off the host: [[vaultwarden-backup-data]]. The data may be migrated by the new version, so the backup is the only way back.
2. Find your current version and how the container was started: [[vaultwarden-find-version-and-deployment]].
3. Read the release notes for every release between the current and target version: https://github.com/dani-garcia/vaultwarden/releases

## If it runs from a compose file
```bash
cd <compose folder>
docker compose pull
docker compose up -d
docker compose logs -f
```
If the compose file pins a version tag, edit the tag first. If it says `latest`, the pull is enough. Older standalone installs use `docker-compose` with a hyphen.

## If it runs from plain docker run
Per the official wiki, preserve the mounted data folder and re-create the container:
```bash
docker pull vaultwarden/server:latest
docker stop <container>
docker rm <container>
docker run -d --name <container> <ALL ORIGINAL FLAGS: data mount, port, networks, env> vaultwarden/server:latest
```
The original flags must be rebuilt from `docker inspect`. Do not guess them. This container is on `reverse-proxy-net` and `security-net`, and it publishes `8080:80`.

## Verify
1. `docker logs <container>` shows a clean start with no errors.
2. `https://<vaultwarden address>/api/config` shows the new `server.version`.
3. Log in on the web vault, the browser extension and the phone app. Confirm sync works.
4. Check the websocket still works (a change on one client appears on another). Reverse proxy settings are in [[vaultwarden-npm-proxy-host]].

## Roll back
Stop the container, restore the backup with [[vaultwarden-restore-backup]], and start the old image tag.

## Notes
- Clients newer than a given release can require a newer server. Seen Oct 3, 2026: client v2026.8.0 and later needs Vaultwarden 1.37.2 or newer.
- Update on a routine, for example monthly, not only when something breaks.

## Related
- [[vaultwarden]]
- [[vaultwarden-backup-data]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Updating-the-vaultwarden-image

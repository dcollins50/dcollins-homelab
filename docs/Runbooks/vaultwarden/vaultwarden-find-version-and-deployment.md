---
name: vaultwarden-find-version-and-deployment
type: runbook
tool: vaultwarden
---

Find out which Vaultwarden version is really running on [[services-host]] and how the container was started. Do this before any backup, update, or config change, because the steps differ for compose and plain `docker run`.

## Why the version is easy to misread

The version shown by the web vault and by the top level `version` field of `/api/config` is the Bitwarden compatible version string (for example `2025.6.0`). Vaultwarden's own version looks like `1.3x.x` and sits under `server.version` in the same response. Found out the hard way on Oct 3, 2026.

## Steps

1. On [[services-host]], print the real version from inside the container:
   ```bash
   for c in $(docker ps --format '{{.Names}}' | grep -i vault); do echo "== $c"; docker exec "$c" /vaultwarden --version; done
   ```
   Add `sudo` before `docker` if it says permission denied.

2. Or read it from the API. Open `https://<vaultwarden address>/api/config` and find `server.version`.

3. Find how it was started, which image and tag it uses, and where its data lives:
   ```bash
   for c in $(docker ps --format '{{.Names}}' | grep -i vault); do echo "== $c"; docker inspect "$c" | grep -E 'compose.project.working_dir|"Image"|"Source"'; done
   ```
   - A `working_dir` line means it was started from a compose file in that folder.
   - No `working_dir` line means it was started with `docker run`. You will need to recreate it from the inspect output.
   - The `Source` line next to a destination of `/data` is the data folder on the host. Back up this folder.

4. List the environment variable names only, never the values, to see which settings are in use:
   ```bash
   for c in $(docker ps --format '{{.Names}}' | grep -i vault); do docker inspect "$c" --format '{{range .Config.Env}}{{println .}}{{end}}' | sed 's/=.*//'; done
   ```

5. Record the results in [[vaultwarden]] (image, tag, data path, compose folder or run flags, setting names).

## Known facts (from earlier sessions)
- Container publishes `8080:80` on [[services-host]].
- Sits on the Docker networks `reverse-proxy-net` and `security-net`.

## Related
- [[vaultwarden]]
- [[vaultwarden-backup-data]]
- [[vaultwarden-update-container]]

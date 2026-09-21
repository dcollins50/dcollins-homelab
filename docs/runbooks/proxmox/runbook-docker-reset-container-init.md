# Runbook: Reset a Docker Container's Init (Stop, Remove, Wipe Volume)

**Category:** Proxmox / Docker-in-LXC
**When to use:** A container was initialized with the wrong settings (wrong name, wrong env vars) and needs to be redone from scratch, where the container writes its config into a named volume at first-run rather than taking it purely from `docker run` flags on every start.

## Why this is necessary

Many init-on-first-run images (step-ca included) only apply `DOCKER_*_INIT_*` environment variables the *first* time they start against an empty volume. Restarting the same container, or even removing and recreating it against the same volume, does not re-trigger initialization — the volume already has state.

## Steps

**1. Stop and remove the container** (the container itself, not the volume):
```bash
docker stop <container-name>
docker rm <container-name>
```

**2. Wipe the volume it initialized into:**
```bash
docker volume rm <volume-name>
```

**3. Re-run with corrected settings.** This time the volume is genuinely empty, so first-run initialization fires again with the new values.

**4. Give the container an explicit name** (`--name`) on the corrected run, rather than letting Docker assign a random one — this avoids a common follow-up mistake of the next command referencing a name that no longer exists because Docker picked a new random one:
```bash
docker run -d --name <chosen-name> -v <volume-name>:<mount-path> ... <image>
```
If you forget `--name` and need to find what Docker actually called it:
```bash
docker ps -a
docker rename <random-name> <chosen-name>
```

## Notes

- Stopping/removing the container does not touch the volume — they're separate objects. This is why step 2 is a distinct command, not implied by step 1.
- If other containers reference this one by name (e.g. compose files, dependent services), update those references after renaming — a stale reference to a since-removed container name fails silently in some tooling rather than erroring clearly.

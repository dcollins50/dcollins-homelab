---
name: nimbus-backup-design
type: decision
status: live
---

## Decision
Built and tested Sep 30, 2026. [[nimbus]] (CT 3400) has two separate backup paths, both ending on the workstation's BitLocker D: drive. The workstation pulls in both cases. The container never pushes.

1. **Container, accounts and database:** a weekly Proxmox backup of CT 3400 to `local` on [[pve-env1]], pulled to `D:\Backups\nimbus`, last 3 kept on both sides.
2. **The files:** a weekly read-only snapshot of the data disk on pve-env1, pulled with rsync to `D:\Backups\cloud-files\current`, with changed and deleted files kept 28 days in `D:\Backups\cloud-files\changed`.

Why two: the Proxmox `local` storage has about 83GB free and the data disk is 250GB with `backup=0`, so the files cannot go through the Proxmox job. See [[cloud-files-design]] for the original reasoning.

## Path 1: container backup
- One Proxmox job covers both [[dev-git]] (CT 3300) and nimbus (CT 3400): `sun 03:00` UTC, storage `local`, snapshot mode, ZSTD, keep last 3. The node runs UTC ([[standardize-node-clocks-utc]]).
- A manual test backup of CT 3400 on Sep 30 was 724,821,989 bytes (about 691 MiB).
- The same pull key and forced command as dev-git are reused. `/usr/local/bin/devgit-backup-send` on pve-env1 now answers `list` and `get` for `vzdump-lxc-3300-*` and `vzdump-lxc-3400-*` files only (one regex, `^vzdump-lxc-(3300|3400)-[0-9_-]+\.tar\.zst$`). The old version is kept as `devgit-backup-send.bak`. The name is misleading now, but renaming it means editing the key line too, so it stayed.
- Workstation: `Scripts\PowerShell\pull-nimbus-backup.ps1`, a copy of the dev-git pull script with the VMID, folder and log name swapped. Scheduled task `Pull nimbus backup`, Sundays at 1pm plus every logon, cloned from the dev-git task. Log: `%LOCALAPPDATA%\nimbus-backup-pull.log`. First pull verified by size against the server's listing. The dev-git pull was re-run after the script change and still works.
- The `.partial` file shows 0 bytes in a Windows directory listing while a download is running. That is Windows not refreshing the size of an open file, not a stall. Read the real size through an open handle if you need progress.

## Path 2: data files
- **Snapshot on pve-env1.** The data disk is an ext4 thin LV (`pve/vm-3400-disk-2`). Inside the unprivileged container the files are owned by uid 100033 on the host with data directory mode 750, so the host can only read them as root. A root cron job, `/etc/cron.d/nimbus-data-snapshot`, runs `/usr/local/bin/nimbus-data-snapshot` at `30 3 * * 0` (Sunday 03:30 UTC, after the Proxmox job). The script puts Nextcloud in maintenance mode, runs `sync`, creates the LV snapshot `pve/snap-nimbus-data`, turns maintenance mode off (a few seconds), then mounts the snapshot read-only at `/mnt/nimbus-data-snap`. Copy of the script: `Scripts\bash\nimbus-data-snapshot.sh`. Log: `/var/log/nimbus-data-snapshot.log`.
- **Restricted key.** `admin` on pve-env1 has a dedicated `nimbus-data-pull` key whose `authorized_keys` line forces `command="sudo /usr/bin/rrsync -ro /mnt/nimbus-data-snap/data",restrict`. `/etc/sudoers.d/nimbus-pull` (mode 440) allows exactly that one command for `admin` and keeps `SSH_ORIGINAL_COMMAND` for `rrsync`. Result: the key can read the snapshot's `data` folder and nothing else, with no shell and no write access. A replaced earlier key line was removed.
- **Workstation pull.** rsync inside WSL2 Ubuntu, script `~/bin/pull-nimbus-data.sh` (copy: `Scripts\bash\pull-nimbus-data.sh`), key `~/.ssh/nimbus_data_pull` with no passphrase because a scheduled task cannot type one. `rsync -rt --delete --backup --backup-dir=.../changed/<date>`, skipping preview and cache folders. Anything changed or deleted lands in `changed\<date>` and is pruned after 28 days. Scheduled task `Pull nimbus data` runs `wsl.exe -d Ubuntu -- /home/[wsl-user]/bin/pull-nimbus-data.sh`, Sundays 1:30pm plus every logon. Log: `%LOCALAPPDATA%\nimbus-data-pull.log`.
- First run copied about 203MB (195MB on disk). A repeat run sent 98 bytes.

## Why these choices
- **Snapshot, not a live copy.** It gives one frozen, consistent state of the files, and the maintenance-mode window is seconds. A snapshot of a thin LV does not need its own reserved space up front.
- **Mirror with a 28-day safety net, not full history.** A plain mirror would copy an accidental delete the next Sunday. Keeping changed and deleted files for 28 days covers that without a second full copy. D: had 887GB free on Sep 29. This is the default I built; Daniel has not said he wants something different.
- **Pull, never push.** Same reasoning as [[dev-git-backup-strategy]]: nimbus sits in the DMZ and the tailnet policy test forbids it from reaching other machines.
- **A plain Nextcloud sync client is not a backup.** Deletions sync too.
- The earlier plan, rclone over WebDAV with an app password, was replaced by this design. The difference that matters is that the files are read from a frozen snapshot instead of through Nextcloud.

## Trade-offs and risks accepted
- **The snapshot can go stale silently.** If the Sunday snapshot job fails, the pull copies last week's snapshot with no error. No freshness check exists yet.
- **The snapshot mount does not survive a pve-env1 reboot.** It returns at the next Sunday run. What the pull does in between has not been tested.
- **Both pulls need the workstation on, Tailscale running and D: unlocked.** A locked D: is logged as `ERROR: D: is not available` on C:. The logon trigger retries.
- **Both pull keys have no passphrase.** Each is locked to a forced command, and the files sit on an encrypted drive.
- **The shared pull key now reaches nimbus container backups as well as dev-git's.** Both are the same kind of data (whole-container backups with application secrets), so this was accepted. It weakens the statement in [[dev-git-backup-strategy]] that the key can only fetch dev-git files.
- **The Proxmox backup of the running database is crash-consistent, not clean.** A nightly database dump inside the container is still an open item.
- **The thin pool is over-provisioned.** `pve/data` was 8.45% used on Sep 30, with a pre-existing warning that total provisioned size exceeds the pool and no autoextend protection. A full pool would hurt every guest on the node.
- **No restore has been tested yet.** Both backup paths are proven to copy, not to restore.

## Open
- Record the results of the read-only tests on the files key (a write attempt, a `../` path, a different command). The restrictions were set up and exercised Sep 30 but the results are not written down here.
- Add a freshness check for the snapshot.
- Test a restore of the container backup and of a file from `current`.

## Related
- [[nimbus]]
- [[cloud-files-design]]
- [[dev-git-backup-strategy]]
- [[pve-env1]]
- [[sop-cloud-files-build]]

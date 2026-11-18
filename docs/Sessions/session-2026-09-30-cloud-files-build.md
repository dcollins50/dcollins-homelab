---
name: session-2026-09-30-cloud-files-build
type: session
date: 2026-09-30
---

Second half of the [[nimbus]] build, after [[session-2026-09-29-cloud-files-planning]]. Current state is in [[nimbus]], backup design in [[nimbus-backup-design]], steps in [[sop-cloud-files-build]].

## What was done
- Fixed the Nextcloud admin overview warnings (APCu, PHP drop-ins, cron background jobs, maintenance window, phone region).
- Switched to HTTPS with `tailscale serve`, closed port 80 in the container firewall, added HSTS, and shared the endpoint to the collaborator. He loaded the HTTPS page.
- Enforced two-factor for all users. Created `collab-user` and `[admin-user]`. the collaborator logged in and set up two-factor.
- Backups built end to end: Proxmox job for CT 3300 and 3400, weekly data-disk snapshot on pve-env1, restricted `rrsync` key, rsync pull in WSL to D:, container backup pull by widening the dev-git forced command. Scheduled tasks `Pull nimbus data` and `Pull nimbus backup` tested with exit code 0.
- Disabled the `[nimbus-admin]` admin account after confirming `[admin-user]` could log in with two-factor.
- Discussed Authentik single sign-on for Nextcloud. Not built. See [[nimbus]].

## What went wrong (for next time)
- I told Daniel the collaborator's page view proved HTTPS worked, but the `tailscale serve` step had never run. Then `overwriteprotocol=https` made the site stop loading. Fix: delete the setting, run the firewall change and `tailscale serve`, check the page, set it again. The SOP step 4 now has the correct order.
- Instructions that said "admin console" without the exact page and click path were not usable. Give the exact URL and click path every time.
- I listed an optional policy edit (removing `tcp:80`) as a step. It is redundant with the container firewall and was skipped.
- Backups: my first idea, reading the data disk from the host, failed because the files are root-only on the host (uid 100033, mode 750) and the host cannot see the container's mount. Redesigned as snapshot plus `rrsync`.
- The pull key had been created outside WSL, so `Identity file ... not accessible` and `Host key verification failed` appeared. A new key was made in WSL and swapped on pve-env1, and the host key was accepted once by hand after comparing the fingerprint.
- The first nimbus container pull looked stuck: the `.partial` file showed 0 bytes for four minutes. I called it a stall. It was not: Windows does not refresh the size of a file that is open for writing, and the server's SSH session was still open. The run was cancelled by accident, rerun, and finished (724,821,989 bytes, size matched).

## Left open at the end of the session
- Nothing on accounts. `[admin-user]` was confirmed logged in with two-factor and `[nimbus-admin]` was disabled at the end of the session.
- Vault write-up was done in this session. Remaining items are listed in [[nimbus]] under Not done yet and in [[nimbus-backup-design]] under Open.

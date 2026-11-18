---
name: proxmox-fix-package-repositories
type: runbook
tool: proxmox
---

# Runbook: Fix a node's Proxmox package repositories

Used on all four nodes Nov 8, 2026. See [[session-2026-11-08-proxmox-9-2-upgrade]].

## Why
Proxmox gets its own updates (kernel, web UI, cluster software) from a Proxmox repository, separate from the Debian ones. Without a working one, `apt` quietly reports nothing to do for Proxmox packages. No subscription is held, so the right repository is **No-Subscription**, on the Debian release the node runs (`trixie` for Proxmox VE 9).

## Check
Click the node in the left tree, then **Updates, Repositories**. Read the Status box at the top.

| Status message | Meaning |
|---|---|
| No Proxmox VE repository is enabled, you do not get any updates! | No usable Proxmox repository. Seen Nov 8 when the only entry was the Proxmox VE 8 `bookworm` one. |
| The enterprise repository is enabled, but there is no active subscription! | The paid repository is on and fails every update check. |
| The Ceph enterprise repository is enabled, but there is no active subscription! | Same for Ceph. Ceph storage is not used here. |
| Some suites are misconfigured | Elastic and Wazuh name their suite `stable` instead of `trixie`. Harmless, leave them. A red `bookworm` in this list is not harmless, see above. |

## Fix
On the same page:
1. Click any row pointing at `enterprise.proxmox.com/debian/pve`, or at `download.proxmox.com/debian/pve` with suite `bookworm`. Click **Disable**.
2. Click any row pointing at `enterprise.proxmox.com/debian/ceph-...`. Click **Disable**.
3. Click **Add**, choose **No-Subscription** in the Repository dropdown, click **Add**.
4. The status should now show only the `stable` suites warning and a note that No-Subscription is not recommended for production. Both are expected.
5. **Updates**, then **Refresh**. It should finish without errors.

Do not click **Upgrade** from here without checking the pending list first. See [[proxmox-upgrade-node]].

## Notes
- Disabling comments the entry out; the old files (`pve-no-subscription.list`, `pve-enterprise.sources`, `ceph.sources`) stay on disk.
- Reference: Proxmox wiki, Package Repositories.

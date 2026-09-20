# Runbook: Troubleshoot Cluster Quorum (Proxmox)

| Field | Value |
| --- | --- |
| Applies to | Proxmox VE (Corosync) |
| Category | Cluster |
| Author | Daniel Collins |

---

## When to use this

Use this when `pvecm status` shows a node (or the whole cluster) as not quorate, or when a node's IP/identity in cluster membership doesn't match reality (e.g. still showing an old address after a network change). This has been a recurring issue in this environment specifically around node reboots after network changes, this isn't hypothetical.

---

## The one fact that matters most here

**`/etc/pve/corosync.conf` is the authoritative, cluster-wide configuration.** It lives on the Proxmox cluster filesystem (pmxcfs) and is automatically synced to every node. `/etc/corosync/corosync.conf` is a local, generated copy, editing it directly does not persist and will be overwritten from the authoritative copy. This has caused real confusion in this environment before, don't edit the local file expecting it to stick.

---

## Diagnosing

1. **Check quorum status:**
   ```bash
   pvecm status
   ```
   Look at `Quorate:` (Yes/No), `Nodes:` (how many are actually participating), and the `Membership information` table for each node's listed address, watch specifically for a node showing a stale/wrong IP.

2. **Check the corosync service itself:**
   ```bash
   systemctl status corosync
   journalctl -u corosync -n 50
   ```
   Distinguish between "corosync is running but the cluster lacks quorum" (a membership/network problem) and "corosync itself failed to start" (a config problem, often a config_version mismatch or syntax error).

3. **Check the actual authoritative config:**
   ```bash
   cat /etc/pve/corosync.conf
   ```
   Confirm each node's listed address matches its real current IP. A stale address here (pointing at an old subnet, an old management IP) is a common root cause, and has been the specific cause of quorum loss in this environment after a node's network config changed.

---

## Fixing a stale node address

1. **Edit the authoritative file, not the local copy:**
   ```bash
   nano /etc/pve/corosync.conf
   ```
2. **Update the affected node's `ring0_addr`** (or equivalent) to its correct current address.
3. **Increment `config_version`** in the `totem` block. Corosync uses this to detect and propagate the newer config, a config change without a version bump may not take effect.
4. Save. Because this file is on the cluster filesystem, the change propagates to other nodes automatically, you generally do not need to manually copy it around.
5. **Restart corosync (and, if needed, the cluster service) on the affected node(s):**
   ```bash
   systemctl restart corosync
   systemctl restart pve-cluster
   ```
6. **Re-check status:**
   ```bash
   pvecm status
   ```
   Confirm `Quorate: Yes` and that the membership table now shows correct addresses for every node.

---

## If a node keeps reverting after every reboot

If the same node's address reverts to a stale value specifically after that node reboots (rather than the fix simply not having been applied), this points to the fix having been made to the wrong file (`/etc/corosync/corosync.conf` instead of `/etc/pve/corosync.conf`) at some point, or to a config that was never actually committed to the cluster filesystem. Re-verify step by step: confirm the authoritative file has the correct value **after** the node has rebooted, not just before, to catch this specific failure mode.

---

## Common mistakes

- **Editing `/etc/corosync/corosync.conf` directly** instead of the cluster-authoritative `/etc/pve/corosync.conf`, the fix appears to work immediately but doesn't survive a restart/reboot.
- **Forgetting to bump `config_version`** after an edit.
- **Not checking whether the root cause is a stale IP versus an actual network/connectivity problem** between nodes, before assuming it's a config file issue. Confirm nodes can actually reach each other on the corosync network first.
- **Restarting corosync cluster-wide unnecessarily** when the issue is isolated to one node's stale entry, a targeted restart on the affected node(s) is usually sufficient.

---

## Related Documentation

- [Infrastructure](../../infrastructure.md)

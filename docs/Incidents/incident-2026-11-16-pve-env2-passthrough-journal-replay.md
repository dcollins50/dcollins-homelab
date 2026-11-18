---
name: incident-2026-11-16-pve-env2-passthrough-journal-replay
type: incident
date: 2026-11-16
status: open
---

# Incident: Host mounted a VM's live passthrough disk, ext4 journal replayed underneath it

**Date:** 2026-11-16 20:48:19 UTC
**Systems affected:** [[pve-env2]], WD PC SN720 256 GB (`nvme1n1`, serial 202118807055), and the VM it is passed through to (600 [[soc-stack-vm|soc-stack]] or 601 [[wazuh-manager]], not yet confirmed)
**Status:** Open.

## What happened

While looking for a disk to use as a local Proxmox Backup Server datastore for the Backblaze B2 offsite plan, the vault listed a 238.5 GB "available" NVMe under [[pve-env1]]. It turned out to be in pve-env2. Its ext4 partition was not a Proxmox storage entry and not mounted on the host, so Claude gave a "read-only" inspection command:

```
sudo mount -o ro /dev/nvme1n1p1 /mnt/peek
```

The filesystem is in live use by a VM (search-engine data layout: `indices/`, `_state/`, `node.lock`, `snapshot_cache/`, 32 GB, written minutes earlier). Because the guest had it mounted, its journal was flagged as needing recovery, and the host kernel replayed the journal onto the disk despite `ro`:

```
EXT4-fs (nvme1n1p1): INFO: recovery required on readonly filesystem
EXT4-fs (nvme1n1p1): write access will be enabled during recovery
EXT4-fs (nvme1n1p1): recovery complete
```

The host wrote to the disk underneath the running guest. The guest's in-memory view of the filesystem and journal no longer matches what is on disk.

## Root cause

- Vault had the drive filed under the wrong node and marked "available".
- Claude did not check VM configs for disk passthrough before mounting.
- `mount -o ro` on ext4 is not guaranteed read-only: it replays a dirty journal. A truly read-only mount needs `-o ro,noload`, and a disk in use by a guest should not be mounted on the host at all.

## Response

Deprioritized by Daniel on Nov 16, 2026; left open. Steps below are pending.

- [ ] Identify which VM has the disk passed through
- [ ] Shut that VM down cleanly
- [ ] `e2fsck -f -n` from the host (VM off), then repair if needed
- [ ] Start the VM, check search-engine cluster health and index status
- [ ] Record outcome here

## Lessons

- Before touching any disk on a node, check `/etc/pve/qemu-server/*.conf` for passthrough (needs sudo).
- Never mount a guest's disk on the host while the guest runs. For inspection with the guest off, use `mount -o ro,noload`.

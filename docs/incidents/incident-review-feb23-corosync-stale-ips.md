# Incident: pve-env1 Failed to Rejoin Quorum After Reboot (Stale corosync.conf IPs)

**Date:** 2026-02-23
**Systems affected:** pve-env1 (10.0.0.12), Proxmox cluster "homelab"
**Status:** Resolved

## Summary

After a clean reboot with no changes beforehand, pve-env1 came back up but showed offline in the cluster. It was reachable by ping and SSH. Corosync was binding to 192.168.100.7, an address from before the management subnet migration to 10.0.0.0/24. The cluster-wide corosync config still listed all four nodes on their old 192.168.100.x addresses. Updating the master config and copying it to the isolated node restored quorum.

## Timeline

Times are from the node's clock, which was set to Hawaii time (HST) instead of Central. See Open items.

- **16:13** Previous boot ends (reboot of pve-env1).
- **16:17** pve-env1 back up. pmxcfs logs `cpg_initialize failed: CS_ERR_LIBRARY`. Corosync starts but forms a one-member ring.
- **16:24** `corosync-cfgtool -s` shows `addr = 192.168.100.7` with nodes 1, 2 and 3 disconnected. `pvecm status` shows `Quorate: No`, 1 of 4 votes, config version 6.
- **~16:30** `/etc/pve/corosync.conf` reviewed on pve-services. All four `ring0_addr` values were old 192.168.100.x addresses.
- **~16:33** Master config edited on pve-services. All four IPs corrected, `config_version` bumped from 6 to 7.
- **16:35** Corosync restarted on pve-env1. Still on version 6, still bound to 192.168.100.7. The update could not sync to a node with no quorum.
- **16:36** Config copied to pve-env1 manually, corosync restarted. 4 of 4 nodes, `Quorate: Yes`, config version 7.

## Root Cause

After the Proxmox management network moved from 192.168.100.x to 10.0.0.x, the cluster-wide `/etc/pve/corosync.conf` was never updated. It still held:

| Node | Stale address | Correct address |
|-|-|-|
| pve-gateway | 192.168.100.2 | 10.0.0.10 |
| pve-services | 192.168.100.3 | 10.0.0.11 |
| pve-env2 | 192.168.100.4 | 10.0.0.13 |
| pve-env1 | 192.168.100.7 | 10.0.0.12 |

Earlier corrections had been made to the local copy at `/etc/corosync/corosync.conf`. That file is generated from `/etc/pve/corosync.conf` by pmxcfs, so those edits were overwritten on sync and the stale addresses came back. When pve-env1 restarted, corosync read the stale address and could not reach any other node.

## Fix

On a quorate node (pve-services), edit the master copy and bump the version:

```bash
sed -i \
  -e 's/ring0_addr: 192.168.100.2/ring0_addr: 10.0.0.10/' \
  -e 's/ring0_addr: 192.168.100.3/ring0_addr: 10.0.0.11/' \
  -e 's/ring0_addr: 192.168.100.7/ring0_addr: 10.0.0.12/' \
  -e 's/ring0_addr: 192.168.100.4/ring0_addr: 10.0.0.13/' \
  -e 's/config_version: 6/config_version: 7/' \
  /etc/pve/corosync.conf
```

A node without quorum cannot receive the update through pmxcfs, so copy it to the isolated node's local file and restart corosync there:

```bash
scp /etc/pve/corosync.conf root@10.0.0.12:/etc/corosync/corosync.conf
systemctl restart corosync
pvecm status
```

(Root SSH was later disabled on all nodes. The same copy now has to go through the `admin` account with sudo.)

## Lessons Learned

- `/etc/pve/corosync.conf` is the only file to edit. `/etc/corosync/corosync.conf` is a generated local copy and edits to it do not stick.
- Always bump `config_version` when editing corosync.conf.
- A node without quorum will not pick up changes from the cluster. It needs the file copied by hand.
- Any management IP change has to include corosync.conf in the same change, or the cluster breaks on the next reboot.

## Open items

- Node timezones were set to Hawaii time (HST/HDT), not Central. This was noticed again on pve-env1 on May 27. It makes log times misleading.

## Related

- A formal runbook (.docx) covering the two-file corosync.conf behavior was written on 2026-03-03.

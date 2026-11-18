---
name: session-2026-11-16-offsite-backups-pbs-b2
type: session
date: 2026-11-16
---

# Offsite backups: Proxmox Backup Server and Backblaze B2

Decision and reasoning: [[offsite-backups-pbs-b2]]. Build details: [[pve-pbs]].

## Done
- Chose Backblaze B2 over Computer Backup (no Linux or server support). Sized the cluster from one SSH session with `pvesh get /cluster/resources`: about 280 GB of guest disk in use, about 221 GB without the lab VMs (300, 301, 302, 400), which are out of scope.
- Found the vault out of date and fixed it: [[pve-ci]] (VM 800) and [[pve-zap]] (LXC 1200) had no notes; [[VLAN55]] had no note; [[ansible-lab]] moved from VMID 1200 to 1101; the 238.5 GB NVMe listed under [[pve-env1]] is actually in [[pve-env2]] and holds the SOC log data.
- Built [[pve-pbs]] (VM 6000, pve-env1, 10.0.60.x on [[VLAN60]], Proxmox Backup Server 4.2) by script. VLAN60 already existed on OPNsense (`vlan07`, tag 60) and the switch.
- Firewall aliases and rules added through the OPNsense API (they show under Firewall > Rules [new]). Listed in [[VLAN60]].
- Datastores: `local-store` (250 GB virtual disk) and `b2-store` (bucket `[b2-bucket]`, region us-east-005, 64 GB cache disk).
- Jobs: nightly backup of all non-lab guests at 08:00 UTC, encrypted on the nodes; sync to B2 at 10:00; prune, garbage collection and verify schedules. Table in [[pve-pbs]].
- Tested: dev-git backed up, restored to throwaway CT 9300 (destroyed afterwards), and synced to B2.

## Problems hit
- Secure Boot violation after the first update on pve-pbs; Secure Boot turned off for that VM.
- A host mount of the SOC log drive on pve-env2 replayed its journal under the running VM. See [[incident-2026-11-16-pve-env2-passthrough-journal-replay]].
- First B2 application key had access to all buckets and full rights; replaced with a key limited to the one bucket.
- A string that was not the B2 keyID was entered at the keyID prompt twice; Daniel confirmed it was not a credential.

## Open
See the Open list in [[pve-pbs]].

---
name: homelab-update-automation
type: project
status: in-progress
---

A new container on [[pve-services]] that runs Ansible through Semaphore. Main job: roll out updates to the four Proxmox nodes and their guests. Second job: take over the Asscord playbooks. Started Nov 7, 2026. Tool choice: [[semaphore-for-update-automation]].

This is separate from [[pve-ansible]], which keeps the Achilles SOC work (see [[achilles-soc-ansible]]). pve-ansible currently also runs the Asscord deploys; those move to ansible-lab, after which pve-ansible's Asscord tailnet grants and leftover files are removed.

## Decided (Nov 7, 2026)
- Container: LXC 1101 (changed from 1200 on Nov 16, 2026; 1200 is now pve-zap on [[pve-gateway]]), hostname [[ansible-lab]], 10.0.30.x on [[VLAN30]] (trust infrastructure), on [[pve-services]].
- LXC updates run from each Proxmox node with `pct exec`, the same way LXCs are managed by hand today (for example dev-git, whose own SSH is off).
- VM updates run from each Proxmox node with `qm guest exec`, which needs the QEMU guest agent in each VM. Daniel believes most or all VMs already have it; not yet checked per VM.
- Web UI access: Daniel only, behind Authentik forward auth.
- Both manual runs and scheduled runs will be explored.
- In scope for updates: the four Proxmox nodes, their guests, and [[heimdall]]. Traffic from Heimdall passes through [[opnsense]], so it is reachable with an OPNsense rule.
- Out of scope: [[metasploitable2]] and [[dvwa]] (meant to stay vulnerable), [[malware-win11]], [[opnsense]] (own update policy), the VPSs except as Asscord deploy targets.

## Asscord playbooks
- Playbooks: on the internal Gitea (services-host), per Daniel (Nov 7, 2026). The `soc-ansible` playbooks are there too.
- [[dev-git]] holds the Asscord and Scrubd source (per Daniel, Nov 8, 2026). the collaborator has not pushed Hermes, Ichos or Achilles there.
- How the deploy works today, confirmed Nov 8, 2026 from the `ansible-asscord` repo README and role defaults on [[pve-ansible]]:
  1. pve-ansible keeps a clone of `[admin-user]/asscord` from dev-git at `/home/ansible/asscord-src`, fetched by hand. It connects over the tailnet to dev-git on 2222 with a read only deploy key (`~/.ssh/asscord_src_deploy`, SSH config host `asscord-src`).
  2. `inventories/staging/group_vars/all.yml` pins the full commit ID to deploy.
  3. The `asscord` role packs exactly that commit from the local clone and pushes it to the VPS over SSH. The VPS never talks to any Gitea.
  - The role finds the clone through `asscord_src_dir` in `roles/asscord/defaults/main.yml`, which defaults to `$HOME/asscord-src` of whoever runs the playbook.
  - The clone's newest commit as of Nov 8, 2026 is `752ae82` (Oct 9, 2026).
- `/home/ansible/asscord-vault-original.yml` is an Ansible Vault encrypted copy (checked Nov 8, 2026), sitting outside the repo. Why it was kept is unknown. Delete after the move works.
- `/home/ansible/asscord-checks.log` is also loose in the home folder. Same cleanup.
- `ansible-lab` on VLAN30 will need a firewall path to the internal Gitea on services-host (VLAN20) for the playbooks.
- Current runner: [[pve-ansible]], which has tailnet grants to the Asscord VPS (SSH) and dev-git (2222).
- They deploy Asscord to a Hostinger VPS used for testing, tagged `tag:asscord-staging` on the homelab tailnet. That VPS is a separate server from the Achilles SOC server.
- Moving the deploys to `ansible-lab` needs the same two grants for ansible-lab, then removal of pve-ansible's two Asscord grants. See [[tailscale-access-model]] and [[tailscale-tailnet-policy]].
- Asscord has no homelab host yet; the "asscord" box on the network diagram stands for that VPS for now.

## pve-services capacity (checked Nov 7, 2026, Proxmox Summary page)
4 cores (i5-7500), RAM 5.76 of 15.50 GiB used, swap 528 MiB of 8 GiB used, root disk 9.85 of 93.93 GiB used. Proxmox VE 9.1.1, kernel 6.17.2-1-pve. The 32 GB RAM upgrade has not happened on this node. local-lvm usage not yet checked.

The Summary page also shows "No Proxmox VE repository enabled!" on this node. Without a Proxmox package repository the node cannot pull Proxmox updates, which this project depends on. Other nodes not yet checked.

## Open items
- Size for `ansible-lab` approved Nov 7, 2026: 2 cores, 2048 MB RAM, 512 MB swap, 20 GB disk. Daniel says local-lvm on pve-services has plenty of room.
- Proxmox package repositories, checked Nov 8, 2026 (node, Updates, Repositories):
  - [[pve-services]] and [[pve-gateway]]: the only Proxmox repository is `pve-no-subscription.list` with suite `bookworm` (the Proxmox VE 8 / Debian 12 repository), on a Proxmox VE 9 / Debian 13 (trixie) node. Left over from the 8 to 9 upgrade. Proxmox reports "No Proxmox VE repository is enabled". No enterprise or Ceph entries.
  - [[pve-env1]] and [[pve-env2]]: both had `pve-enterprise.sources` and `ceph.sources` (ceph-squid enterprise) enabled on `trixie` with no subscription, and no No-Subscription repository. Both fixed Nov 8, 2026.
  - pve-env1 pending after refresh: 108 Proxmox packages, 32 Debian (same security-relevant set as pve-gateway, plus jq), wazuh-agent 4.14.7 to 4.14.10, tailscale 1.104.2 to 1.104.5, filebeat 8.19.21 to 8.19.24.
  - pve-env2 pending after refresh: 108 Proxmox packages, 30 Debian, wazuh-agent 4.14.8 to 4.14.10, tailscale 1.104.2 to 1.104.5.
  - corosync is 3.1.9 on pve-env1 and pve-env2 and 3.1.10 on pve-gateway and pve-services today; all go to 3.2.0.
  - All four nodes now have a working No-Subscription repository and the same Proxmox VE 9.2 upgrade pending.
  - All nodes seen also carry the Elastic 8.x and Wazuh 4.x repositories on suite `stable`, which Proxmox flags as misconfigured. That is a naming mismatch on third-party repositories, not a fault.
  - Fix: disable the stale or paid entries, add No-Subscription for trixie, refresh, then review pending updates before any upgrade.
  - pve-gateway fixed Nov 8, 2026. Pending after refresh: the same 109 Proxmox packages, 30 Debian packages (including openssl and libssl3 3.5.7, curl, bind9 tools, libexpat1, rsync 3.4.1 to 3.5.0, tzdata), wazuh-agent 4.14.7 to 4.14.10, tailscale 1.104.2 to 1.104.5, filebeat 8.19.21 to 8.19.24. Its Debian packages are further behind than pve-services'.
  - pve-services fixed Nov 8, 2026. Pending after refresh: 109 Proxmox packages, 8 Debian, plus wazuh-agent 4.14.9 to 4.14.10, tailscale 1.104.4 to 1.104.5, filebeat 8.19.23 to 8.19.24.
- The pending Proxmox packages are the Proxmox VE 9.1 to 9.2 upgrade (pve-manager 9.1.1 to 9.2.26), not routine patches. 9.2 (released May 21, 2026) makes kernel 7.0 the default and brings QEMU 11, LXC 7.0 (lxc-pve 6.0.5 to 7.0.0) and ZFS 2.4, plus corosync 3.1.10 to 3.2.0. Needs a reboot per node for the new kernel; running VMs keep the old QEMU until stopped and started.
  - Forum reports exist of kernel 7.0 problems on some hardware (host freezes, Windows Server 2025 VMs failing to boot). The 6.17 kernel stays installed (6.17.13-21 is in the list), so it can be pinned as a fallback.
  - Order decided Nov 8, 2026: pve-gateway first, then pve-env1, pve-env2, pve-services last. Done by hand, one node at a time.
  - Decided Nov 8, 2026: the SOC pipeline is updated separately, not in this round. On every node, `wazuh-agent` and `filebeat` are held (`apt-mark hold`) during the Proxmox upgrade. They get updated together with wazuh-manager and soc-stack later.
  - [[malware-win11]] has nothing on it yet and can be rebuilt, so no backup before the pve-gateway reboot.
  - Nov 8, 2026: `apt full-upgrade` run on all four nodes the same night (not one node at a time as planned), with `wazuh-agent` and `filebeat` held. Answered No to the shim "Disable UEFI Secure Boot?" prompt on pve-gateway. Packages are on Proxmox VE 9.2; reboots onto kernel 7.0 not yet done as of that report. Reboots still go one node at a time in the decided order.
  - Post-upgrade check Nov 8, 2026, all four nodes before any reboot: Quorate Yes, 4 of 4 votes, `pve-manager/9.2.26`, running kernel `6.17.2-1-pve`, kernels installed `6.17.2-1-pve`, `6.17.13-21-pve`, `7.0.14-28-pve` (none pinned, so the next boot picks 7.0), `systemctl --failed` empty, holds on wazuh-agent and filebeat confirmed. Fallback kernel if 7.0 misbehaves: `6.17.2-1-pve`, the one each node has actually been running.
  - [[pve-gateway]] rebooted Nov 8, 2026: running `7.0.14-28-pve`, `pve-manager/9.2.26`, Quorate Yes with 4 of 4 votes, no failed units. malware-win11 boot on the new QEMU not yet reported.
  - [[pve-env1]] rebooted Nov 8, 2026: running `7.0.14-28-pve`, `pve-manager/9.2.26`, Quorate Yes with 4 of 4 votes, no failed units. Whether all its guests started on their own not yet confirmed.
  - [[pve-env2]] and [[pve-services]] rebooted Nov 8, 2026, without the pve-services guest backups. Daniel reports all nodes and guests back online; no command output checked for these two.
  - Left over from this round: `wazuh-agent` and `filebeat` still held on all four nodes (release with `apt-mark unhold` in the SOC session); the disabled `bookworm` and enterprise repository entries still sit in their files. Confirmed by Daniel Nov 9, 2026: the pve-services NVMe workaround is still on the kernel 7.0 command line, SOC log flow after the pve-env2 reboot, malware-win11 boots and logs in, all guests that need Start at boot have it.
  - pve-services guests still have no backups other than the Oct 17 one-off copies. Tabled Nov 9, 2026 as its own project, not part of this one.
- How pve-ansible reaches the Achilles server: Daniel believes SSH over a tailnet. Its host note says a separate SOC tailnet, but the live homelab policy shows pve-ansible on the homelab tailnet (100.x.x.x) with no grant to achilles. Not resolved, not blocking this project.
- Confirm the QEMU guest agent per VM (`qm config` and `qm agent <vmid> ping` on each node).
- The account Semaphore uses on each Proxmox node (root SSH is off on all four), and its sudo rule. `pct exec` and `qm guest exec` give root inside every guest, so this account is root over the cluster in practice.
- Key handling: pve-ansible uses a typed passphrase and typed Vault password each run plus signed commit checks. Scheduled runs cannot type anything.
- Tailscale on `ansible-lab`: TUN device passthrough (same as [[ansible-setup-control-node]] step 2), a tag, and a tailnet policy grant limited to the Asscord VPS on SSH, with a policy test. Record it as an exception in [[tailscale-access-model]].
- Reboot policy: automatic one node at a time, or report only.
- Firewall rules from VLAN30 to the node management addresses, Heimdall, Gitea, package mirrors, and from the internal NPM / Authentik outpost to the Semaphore UI.

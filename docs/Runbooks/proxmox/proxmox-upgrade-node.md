---
name: proxmox-upgrade-node
type: runbook
tool: proxmox
---

# Runbook: Upgrade and reboot a Proxmox node

Used for the 9.1 to 9.2 upgrade on Nov 8, 2026. See [[session-2026-11-08-proxmox-9-2-upgrade]]. The repositories must be fixed first: [[proxmox-fix-package-repositories]].

Run from the workstation's direct connection, not over Tailscale (the node's Tailscale package may restart during the upgrade). Commands run in the node's **Shell** as `admin` with sudo.

## 1. Cluster healthy
```
sudo pvecm status
```
Need **Quorate: Yes** and **Total votes: 4**. Otherwise stop.

## 2. Hold packages that belong to another update
The SOC pipeline is updated separately, so on every node:
```
sudo apt-mark hold wazuh-agent filebeat
apt-mark showhold
```
Release later with `sudo apt-mark unhold wazuh-agent filebeat`.

## 3. Upgrade
```
sudo apt update
sudo apt full-upgrade
```
Always `full-upgrade`, never plain `apt upgrade`, on a Proxmox node. Before answering **Y**, check the summary: held packages listed as held back, and nothing under **will be REMOVED**.

During the run:
- Changelog pager: press **q**.
- Configuration file prompt: stop and decide deliberately; don't guess.
- shim "Disable UEFI Secure Boot?" prompt: answer **No**. The Proxmox kernels come signed. Yes would turn off boot signature checking and require a one-time password at the physical console.

## 4. Before rebooting
```
sudo pvecm status
sudo proxmox-boot-tool kernel list
pveversion
systemctl --failed
```
The new kernel should be listed, pveversion shows the new version with the old running kernel, and no failed units.

## 5. Reboot, one node at a time
Node page, **Reboot**. A node reboot shuts down all its guests. With four nodes, quorum (3 votes) holds while one is down. Wait until the node is back and checked before rebooting the next.

Order used Nov 8, 2026: [[pve-gateway]], [[pve-env1]], [[pve-env2]], [[pve-services]] (least impact first; pve-services carries Authentik, step-ca and the bastion).

## 6. After
```
uname -r
pveversion
sudo pvecm status
systemctl --failed
sudo pct list
sudo qm list
```
Then confirm guests that should run are running (any without Start at boot stay off).

## If the node won't come back on the new kernel
At the machine with a monitor and keyboard, choose the previously running kernel in the boot menu (Advanced options). Once up, pin it before touching any other node:
```
sudo proxmox-boot-tool kernel pin <version>
```
On Nov 8, 2026 the known-good fallback was `6.17.2-1-pve`.

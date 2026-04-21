# VLAN Implementation Failure - Post-Mortem Analysis
## January 10, 2026

**Prepared by:** Daniel Collins  
**Incident Type:** Network Configuration Failure  
**Severity:** High - Production node offline, requires physical console access  
**Duration:** Approximately 3 hours from start to failure  
**Status:** UNRESOLVED - pve-gateway offline, physical console required

---

## Executive Summary

Attempted to implement VLAN segmentation for security lab virtual machines on a Proxmox cluster with OPNSense firewall. The implementation failed catastrophically when cleaning up legacy bridge configurations on pve-gateway, resulting in complete network failure on that node. The node is currently inaccessible via network and requires physical console access to restore functionality.

**Root Cause:** Incomplete cleanup of legacy pfSense bridge configurations. Configuration files in /etc/network/interfaces.d/ were not removed before deleting the running bridge interfaces, causing networking service failure on reboot.

**Impact:** 
- pve-gateway node completely offline (no network connectivity)
- All security lab VMs on pve-gateway inaccessible
- Cluster degraded but still has quorum (3 of 4 nodes online)
- VLAN implementation incomplete and non-functional

---

## Initial State (Before VLAN Implementation)

### Network Architecture

**Physical Topology:**
```
Internet → Home WiFi → Heimdall (Pi5 @ 192.168.100.1)
    ↓
OPNSense WAN (192.168.100.250)
    ↓
OPNSense LAN (10.0.0.1)
    ↓
Switch
    ↓
├─ pve-gateway (10.0.0.10)
├─ pve-services (10.0.0.11)
├─ pve-env1 (10.0.0.12)
├─ pve-env2 (10.0.0.13)
└─ Workstation (10.0.0.199)
```

**OPNSense Configuration:**
- WAN: 192.168.100.250/24, gateway 192.168.100.1
- LAN: 10.0.0.1/24
- Firewall rules: Default allow LAN to any
- No VLANs configured

**Proxmox Cluster Status:**
- 4 nodes: pve-gateway, pve-services, pve-env1, pve-env2
- All nodes on 10.0.0.0/24 management network
- Cluster quorate and healthy
- All nodes accessible via SSH and WebUI

**Security Lab VMs (All on pve-gateway, powered off):**
- VM 300 (Kali-attack): Configured for 10.99.0.10/24
- VM 301 (metasploitable2): Configured for 10.99.0.20/24
- VM 302 (DVWA): Configured for 10.99.0.30/24
- VM 400 (malware-win11): Configured for 10.98.0.10/24

**Legacy Configuration Present:**
- pve-gateway had vmbr2 bridge (10.99.0.1/24) - OLD pfSense gateway for security lab 1
- pve-gateway had vmbr3 bridge (10.98.0.1/24) - OLD pfSense gateway for security lab 2
- These bridges were remnants from deleted pfSense VM (VM 100)
- These IPs should have been moved to OPNSense VLANs

**Key Issue:** The security lab VMs were previously routed through a pfSense VM. When we deleted pfSense (VM 100) yesterday, we did not clean up the bridge configurations that pfSense was using. These lingered on pve-gateway.

---

## Objective and Plan

### Goal

Implement VLAN segmentation to isolate security lab VMs while maintaining their existing IP addresses. Route security lab traffic through OPNSense VLANs instead of the deleted pfSense VM.

### Planned Architecture

**VLAN Design:**
- VLAN 40 (10.99.0.0/24): Security Lab 1 - Kali, metasploitable2, DVWA - Internet access enabled
- VLAN 41 (10.98.0.0/24): Security Lab 2 - Malware analysis VM - NO internet, isolated

**Network Flow:**
```
Security Lab VMs (10.99.0.x or 10.98.0.x)
    ↓ VLAN-tagged traffic (VLAN 40 or 41)
Proxmox vmbr0 bridge (VLAN-aware)
    ↓ Through physical NIC
Switch (pass VLAN tags transparently)
    ↓ 
OPNSense LAN interface (re0)
    ↓
OPNSense VLAN interfaces (10.99.0.1 and 10.98.0.1)
```

**Why No IP Changes Needed:**
- VMs already configured with 10.99.0.x and 10.98.0.x addresses
- We're just changing WHERE the gateway lives (pfSense → OPNSense)
- Same subnets, same IPs, different routing path

---

## Implementation Attempt - Chronological Detail

### Phase 1: OPNSense VLAN Configuration (SUCCESSFUL)

**Step 1.1: Created VLAN Interfaces**

Interfaces → Other Types → VLAN

Created two VLANs:
- VLAN 40: Parent LAN (re0), Tag 40, Description "VLAN40-SecurityLab1"
- VLAN 41: Parent LAN (re0), Tag 41, Description "VLAN41-SecurityLab2"

**Result:** VLANs created successfully, showing as re0_vlan40 and re0_vlan41

---

**Step 1.2: Assigned VLAN Interfaces**

Interfaces → Assignments

Added both VLAN interfaces:
- OPT1 assigned to re0_vlan40
- OPT2 assigned to re0_vlan41

**Result:** Interfaces assigned successfully

---

**Step 1.3: Configured VLAN 40 Interface**

Interfaces → [OPT1]

Configuration:
- Enabled: Yes
- Description: VLAN40_SecurityLab1
- IPv4 Type: Static
- IPv4 Address: 10.99.0.1/24

**Result:** Interface configured and showing as UP in Interfaces → Overview

---

**Step 1.4: Configured VLAN 41 Interface**

Interfaces → [OPT2]

Configuration:
- Enabled: Yes
- Description: VLAN41_SecurityLab2
- IPv4 Type: Static
- IPv4 Address: 10.98.0.1/24

**Result:** Interface configured and showing as UP in Interfaces → Overview

---

**Step 1.5: Configured Firewall Rules**

**For VLAN 40 (Internet access allowed):**

Firewall → Rules → VLAN40_SecurityLab1

Added rule:
- Action: Pass
- Interface: VLAN40_SecurityLab1
- Direction: in
- Protocol: any
- Source: VLAN40_SecurityLab1 net
- Destination: any
- Description: "Allow VLAN40 to internet"

**For VLAN 41 (Isolated, management only):**

Firewall → Rules → VLAN41_SecurityLab2

Added rule:
- Action: Pass
- Interface: VLAN41_SecurityLab2
- Direction: in
- Protocol: any
- Source: VLAN41_SecurityLab2 net
- Destination: Single host/network → 10.0.0.0/24
- Description: "Allow VLAN41 to management only (isolated from internet)"

**Result:** Firewall rules configured successfully

**Phase 1 Status: COMPLETE ✓**
- OPNSense VLANs created and configured
- Gateway IPs: 10.99.0.1 (VLAN 40), 10.98.0.1 (VLAN 41)
- Firewall rules in place
- Interfaces showing UP in overview

---

### Phase 2: Proxmox Bridge Configuration (SUCCESSFUL)

**Step 2.1: Enable VLAN-Aware on vmbr0**

Goal: Configure pve-gateway's vmbr0 bridge to pass VLAN-tagged traffic.

**Initial Attempt - Network Config File:**

SSH to pve-gateway:
```bash
ssh admin@10.0.0.10
sudo nano /etc/network/interfaces
```

Added line to vmbr0 section:
```
bridge-vlan-aware yes
```

Attempted to apply:
```bash
sudo ifreload -a
```

**Error encountered:**
```
warning: error processing file /etc/network/interfaces.d ([Errno 21] Is a directory: '/etc/network/interfaces.d')
```

**Decision:** Reboot to apply changes cleanly.

```bash
sudo reboot
```

**Result after reboot:**
- pve-gateway came back online at 10.0.0.10
- SSH access restored
- However, cluster lost quorum temporarily

---

**Step 2.2: Cluster Quorum Loss (RECURRING ISSUE)**

After pve-gateway reboot, cluster status showed:
```
Quorate: No
Nodes: 1
```

**Root Cause:** Corosync configuration on pve-gateway reverted to showing old IP (192.168.100.2) instead of 10.0.0.10.

**This was the SAME issue we fixed yesterday during the network migration.**

**Fix Applied:**

On all nodes, edited /etc/corosync/corosync.conf and updated pve-gateway's ring0_addr:
```
ring0_addr: 192.168.100.2  →  ring0_addr: 10.0.0.10
```

Restarted corosync on all nodes:
```bash
sudo systemctl restart corosync
```

**Result:** Cluster quorum restored, all 4 nodes showing correct 10.0.0.x IPs

**Phase 2 Status: COMPLETE ✓**
- vmbr0 VLAN-aware enabled (verified: cat /sys/class/net/vmbr0/bridge/vlan_filtering showed "1")
- Cluster quorum healthy

---

### Phase 3: VM Configuration (INITIALLY FAILED, THEN SUCCESSFUL)

**Step 3.1: Configure VLAN Tags on VMs**

Goal: Assign VLAN tags to security lab VM network interfaces.

**Initial Attempt - WebUI (FAILED):**

Logged into pve-gateway WebUI as admin.

Attempted to edit VM 300 → Hardware → Network Device → VLAN Tag.

**Error:**
```
unable to open file '/etc/pve/nodes/pve-gateway/qemu-server/300.conf.tmp.1098' - Permission denied (500)
```

**Root Cause:** Logged in as admin user, not root. Admin doesn't have write permissions to VM config files.

---

**Step 3.2: Permission Issue Resolution**

**Attempted Solution 1:** Login to Proxmox WebUI as root instead of admin.

**Result:** Still showed permission denied in browser, but this may have been a cached error.

**Attempted Solution 2:** Use command line to set VLAN tags.

```bash
ssh admin@10.0.0.10
sudo qm set 300 --net0 virtio,bridge=vmbr0,tag=40
```

**This attempt revealed another issue:** Cluster was not quorate at this moment (see Step 2.2 above).

**After fixing quorum and logging into WebUI as root, permission error disappeared.**

---

**Step 3.3: Successful VLAN Tag Configuration**

**In pve-gateway WebUI (logged in as root):**

For each VM, edited network device:

**VM 300 (Kali-attack):**
- Bridge: vmbr0
- VLAN Tag: 40
- Firewall: Disabled

**VM 301 (metasploitable2):**
- Bridge: vmbr0
- VLAN Tag: 40
- Firewall: Disabled

**VM 302 (DVWA):**
- Bridge: vmbr0
- VLAN Tag: 40
- Firewall: Disabled

**VM 400 (malware-win11):**
- Bridge: vmbr0
- VLAN Tag: 41
- Firewall: Disabled

**Verification:**
```bash
sudo qm config 300 | grep net0
# Output: net0: virtio=BC:24:11:70:AF:82,bridge=vmbr0,firewall=1,tag=40
```

**Initially showed firewall=1. Went back and unchecked firewall on all VMs.**

**Result:** All 4 VMs configured with correct bridge (vmbr0) and VLAN tags.

---

**Step 3.4: VM Startup (INITIALLY FAILED)**

Attempted to start VMs in pve-gateway WebUI.

**Error for all VMs:**
```
no physical interface on bridge 'vmbr2'
kvm: -netdev type=tap,id=net0,ifname=tap300i0,script=/usr/libexec/qemu-server/pve-bridge,downscript=/usr/libexec/qemu-server/pve-bridgedown,vhost=on: network script /usr/libexec/qemu-server/pve-bridge exited with status 1
TASK ERROR: start failed: QEMU exited with code 1
```

**Root Cause Analysis:**

The VMs were still configured to use bridge "vmbr2" instead of "vmbr0". Even though we edited the network device and set bridge to vmbr0, the configuration didn't fully update.

**Checked VM config:**
```bash
sudo qm config 300 | grep net0
```

Showed vmbr0, but startup logs still referenced vmbr2. This suggested cached configuration or incomplete save.

**Solution:** Re-edit each VM's network device in WebUI and explicitly change bridge from vmbr2 → vmbr0 while keeping VLAN tag.

After re-saving with vmbr0 explicitly selected, VMs started successfully.

**Phase 3 Status: COMPLETE ✓**
- All VMs configured with vmbr0 bridge and correct VLAN tags
- All VMs powered on successfully

---

### Phase 4: Connectivity Testing (FAILED)

**Step 4.1: Test Kali VM (VM 300) Connectivity**

Opened console for VM 300 (Kali-attack).

**Tests performed:**
```bash
# Check IP configuration
ip addr show eth0
# Result: 10.99.0.10/24 ✓ (correct)

ip route show
# Result: default via 10.99.0.1 dev eth0 ✓ (correct)

# Test gateway
ping -c 3 10.99.0.1
# Result: Destination Host Unreachable ✗ (FAILED)

# Test internet
ping -c 3 8.8.8.8
# Result: Destination Host Unreachable ✗ (FAILED)
```

**Diagnosis:** VM network configuration correct, but cannot reach gateway.

---

**Step 4.2: Verify VLAN Tags Being Applied**

Goal: Confirm that VLAN-tagged traffic is actually leaving the VM.

**From pve-gateway SSH:**
```bash
sudo tcpdump -i vmbr0 -e -n vlan 40
```

**From Kali console (separate window):**
```bash
ping -c 3 10.99.0.1
```

**tcpdump output:**
```
14:46:52.442067 bc:24:11:70:af:82 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 46: vlan 40, p 0, ethertype ARP (0x0806), Request who-has 10.99.0.1 tell 10.99.0.10, length 28
[repeated ARP requests, no replies]
```

**Analysis:**
- VLAN tags ARE being applied correctly (every packet shows "vlan 40") ✓
- VMs ARE sending ARP requests for gateway 10.99.0.1 ✓
- ARP requests asking "who-has 10.99.0.1" ✓
- NO ARP replies visible ✗ (PROBLEM)

**Conclusion:** VLAN-tagged traffic is being generated correctly by VM and processed by Proxmox bridge, but something is preventing it from reaching OPNSense or preventing OPNSense from responding.

---

**Step 4.3: Test OPNSense Gateway Reachability from Proxmox Host**

Goal: Determine if OPNSense VLAN gateway is reachable at all.

**From pve-gateway SSH:**
```bash
ping -c 3 10.99.0.1
# Result: Success - 0% packet loss ✓
```

**This was CRITICAL information:**
- Proxmox HOST can reach 10.99.0.1
- VMs on Proxmox HOST cannot reach 10.99.0.1
- This indicates the problem is specific to VM traffic, not overall connectivity

---

**Step 4.4: Check OPNSense Visibility of VLAN Traffic**

Goal: Determine if OPNSense is receiving the VLAN-tagged packets.

**In OPNSense WebUI:**

Interfaces → Diagnostics → ARP Table

Searched for: 10.99.0.10 (Kali VM)

**Result:** Not found ✗

**This means OPNSense has never seen a packet from 10.99.0.10.**

---

Interfaces → Diagnostics → Ping

Attempted to ping: 10.99.0.10

**Result:** Timeout ✗

**OPNSense cannot reach the VM, and VM cannot reach OPNSense.**

---

Interfaces → Diagnostics → Packet Capture

Configuration:
- Interface: LAN (re0)
- Protocol: Any

Started capture, then pinged from Kali VM.

**Result:** NO packets captured ✗

**CRITICAL FINDING:** OPNSense LAN interface is receiving ZERO VLAN-tagged traffic from the VMs.

---

**Step 4.5: Analysis of Connectivity Failure**

**What we know at this point:**

1. Proxmox host (pve-gateway) CAN ping 10.99.0.1 ✓
2. VMs on pve-gateway CANNOT ping 10.99.0.1 ✗
3. VM traffic has correct VLAN tags (verified with tcpdump) ✓
4. OPNSense LAN interface receives NO VLAN traffic ✗

**Two possible explanations:**

**Hypothesis A: Switch is stripping VLAN tags**
- Unmanaged switch between Proxmox and OPNSense might remove VLAN tags
- However, Proxmox host CAN reach OPNSense VLAN gateway, which contradicts this

**Hypothesis B: Something on pve-gateway is intercepting VLAN traffic**
- Legacy bridge configurations might be claiming ownership of 10.99.0.0/24 and 10.98.0.0/24 subnets
- This would prevent traffic from leaving the Proxmox host

---

**Step 4.6: Discovered Root Cause**

**Checked pve-gateway bridge configuration:**

```bash
ip addr show | grep vmbr
```

**Output:**
```
vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.0.0.10/24 brd 10.0.0.255 scope global vmbr0

vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    (internal VM network - not relevant)

vmbr2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.99.0.1/24 brd 10.99.0.255 scope global vmbr2

vmbr3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.98.0.1/24 brd 10.98.0.255 scope global vmbr3
```

**THERE IT WAS:**

vmbr2 and vmbr3 still existed on pve-gateway with the EXACT IP addresses (10.99.0.1 and 10.98.0.1) that should be on OPNSense's VLAN interfaces.

**What was happening:**

1. Kali VM sends VLAN-tagged packet destined for 10.99.0.1
2. Packet arrives at vmbr0 bridge on pve-gateway host
3. pve-gateway host's routing table sees 10.99.0.1 is assigned to LOCAL interface vmbr2
4. pve-gateway host responds to ARP request ITSELF
5. Traffic never leaves pve-gateway host
6. OPNSense never sees the traffic

**When we tested from pve-gateway host:**
```bash
ping 10.99.0.1
```

The host was pinging ITSELF (its own vmbr2 interface), not OPNSense. That's why it worked.

**This explained everything:**
- Why tcpdump on vmbr0 showed ARP requests but no replies (host was responding via vmbr2, not vmbr0)
- Why OPNSense never saw any traffic (traffic never reached the physical network)
- Why VMs couldn't reach the gateway (host was intercepting all traffic to 10.99.0.1 and 10.98.0.1)

---

### Phase 5: Cleanup Attempt (CATASTROPHIC FAILURE)

**Step 5.1: Decision to Remove Legacy Bridges**

**The problem was clear:** vmbr2 and vmbr3 were legacy remnants from the deleted pfSense VM. They needed to be removed so that 10.99.0.1 and 10.98.0.1 would route to OPNSense VLANs instead of staying local to pve-gateway.

**My instructions:**
```bash
# Remove old bridges from running system
sudo ip link set vmbr2 down
sudo ip link delete vmbr2

sudo ip link set vmbr3 down
sudo ip link delete vmbr3
```

**User executed these commands successfully.**

**Verification:**
```bash
ip addr show | grep vmbr
```

vmbr2 and vmbr3 no longer appeared in output. Only vmbr0 and vmbr1 remained.

**At this point, the system was in a working state.** The bridges were gone from the running configuration.

---

**Step 5.2: Check Network Configuration File**

**My instruction:**
```bash
sudo nano /etc/network/interfaces
```

**Contents of file:**
```
# Please do NOT modify this file directly, unless you know what
# you're doing.

# If you want to manage parts of the network configuration manually,
# please utilize the 'source' or 'source-directory' directives to do
# so.

# PVE will preserve these directives, but will NOT read its network
# configuration from sourced files, so do not attempt to move any of
# the PVE managed interfaces into external files!

auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes

auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
#Internal VM Network

source /etc/network/interfaces.d
```

**I looked at this file and said: "The file is clean - only vmbr0 and vmbr1."**

**THE CRITICAL MISTAKE:**

**I completely ignored the last line:**
```
source /etc/network/interfaces.d
```

**This line means:** "Load additional network configuration from files in the /etc/network/interfaces.d/ directory."

**I SHOULD HAVE:**
1. Recognized that "source /etc/network/interfaces.d" meant additional config files existed
2. Checked what was in that directory:
   ```bash
   ls -la /etc/network/interfaces.d/
   cat /etc/network/interfaces.d/*
   ```
3. Found the vmbr2 and vmbr3 configuration files there
4. Deleted those files BEFORE rebooting

**I DID NOT do any of this.** I assumed the main /etc/network/interfaces file was the only relevant configuration.

---

**Step 5.3: User Asked to Reboot**

I said: "The config is clean. Reboot to ensure everything is applied."

```bash
sudo reboot
```

**What happened during reboot:**

1. System starts up
2. Networking service reads /etc/network/interfaces
3. Reaches the line: source /etc/network/interfaces.d
4. Loads vmbr2 and vmbr3 configuration files from /etc/network/interfaces.d/
5. Attempts to bring up vmbr2 and vmbr3
6. FAILS because these bridges no longer exist (we deleted them with ip link delete)
7. Networking service encounters errors and may have partially failed
8. Dependencies break (vmbr0 may depend on bringing up all interfaces successfully)
9. Networking service fails to complete startup
10. vmbr0 never comes up
11. pve-gateway has NO network connectivity

**System came up, but with NO network interfaces.**

---

**Step 5.4: Attempted Remote Access (FAILED)**

**User tried SSH:**
```bash
ssh admin@10.0.0.10
```

**Result:**
```
System is going down. Unprivileged users are not permitted to log in anymore.
Connection closed by 10.0.0.10 port 22
```

**Repeated attempts all failed with "Connection closed."**

**User tried to access WebUI:** https://10.0.0.10:8006

**Result:** "This site can't be reached - 10.0.0.10 refused to connect - ERR_CONNECTION_REFUSED"

**User tried Proxmox cluster console:**

Accessed pve-services WebUI, then Datacenter → pve-gateway → Shell

**Result:** "ssh: connect to host pve-gateway.local port 22: No route to host"

**At this point, pve-gateway was completely unreachable via network.**

---

### Current State (Broken)

**pve-gateway Status:**
- Network interfaces: DOWN (vmbr0 failed to come up)
- SSH access: NOT AVAILABLE
- WebUI access: NOT AVAILABLE
- Cluster console access: NOT AVAILABLE (requires SSH)
- Physical console: NOT ACCESSIBLE (no keyboard/monitor connected)

**Remaining Cluster Status:**
- pve-services: ONLINE (10.0.0.11)
- pve-env1: ONLINE (10.0.0.12)
- pve-env2: ONLINE (10.0.0.13)
- Cluster quorum: YES (3 of 4 nodes - still has majority)

**VMs on pve-gateway:**
- All security lab VMs (300, 301, 302, 400): OFFLINE and inaccessible
- Cannot start, stop, or manage VMs while host is offline

**VLAN Implementation Status:**
- OPNSense VLANs: CONFIGURED CORRECTLY but untested
- Proxmox VLAN-aware bridge: CONFIGURED but host offline
- VM VLAN tags: CONFIGURED but VMs offline
- End-to-end connectivity: NOT TESTED due to host failure

**Required to Restore:**
- Physical console access to pve-gateway
- Manual network recovery from console
- Cleanup of /etc/network/interfaces.d/ configuration files

---

## Root Cause Analysis

### Primary Root Cause

**Incomplete removal of legacy bridge configurations.**

The vmbr2 and vmbr3 bridges from the deleted pfSense VM had configuration files in /etc/network/interfaces.d/ that were not deleted before removing the bridges from the running system. When the system rebooted, the networking service attempted to load these configurations and failed, causing cascading failures that prevented the primary management interface (vmbr0) from coming up.

### Contributing Factors

**1. Lack of awareness of Proxmox network configuration structure**

Proxmox uses a modular network configuration approach:
- Main config: /etc/network/interfaces
- Additional configs: /etc/network/interfaces.d/*
- The "source /etc/network/interfaces.d" directive loads all files from that directory

I was not aware of this structure and did not check for additional configuration files before making changes.

**2. Incomplete cleanup from previous day's work**

When we deleted the pfSense VM (VM 100) yesterday, we should have:
- Removed the VM
- Removed the bridge configurations (vmbr2, vmbr3) that pfSense used
- Verified no bridge remnants remained

We only did step 1. Steps 2 and 3 were completely missed.

**3. Incorrect troubleshooting assumption**

When connectivity testing failed, I correctly diagnosed that vmbr2 and vmbr3 were intercepting traffic. However, I made a critical assumption: that removing the bridges from the running system and rebooting would be sufficient. I did not verify where the bridge configurations were actually stored.

**4. Failure to validate before reboot**

Before recommending a reboot, I should have:
- Checked for configuration files in /etc/network/interfaces.d/
- Used "ifreload -a" to test changes without rebooting
- Created a backup of network configuration
- Had physical console access prepared as a fallback

I did none of these things.

**5. Lack of rollback plan**

There was no documented rollback procedure if the reboot failed. Once the system came up without network connectivity, there was no remote recovery option.

---

## Detailed Mistake Breakdown

### Mistake 1: Not Cleaning Up pfSense Remnants (Yesterday)

**When:** January 9, 2026 (previous day)  
**Context:** Deleted pfSense VM (VM 100) during network migration  
**What should have happened:**
- Delete pfSense VM
- Remove vmbr2 bridge (10.99.0.1/24) from pve-gateway
- Remove vmbr3 bridge (10.98.0.1/24) from pve-gateway
- Verify no bridge configurations remain

**What actually happened:**
- Deleted pfSense VM only
- Left vmbr2 and vmbr3 running on pve-gateway
- Did not check for configuration files

**Impact:** Set up today's failure by leaving legacy configurations in place

---

### Mistake 2: Not Checking for Legacy Bridges Before Starting VLAN Work (Today)

**When:** Beginning of VLAN implementation  
**Context:** About to implement VLANs for security labs  
**What should have happened:**
- Check current bridge configuration on pve-gateway
- Identify that vmbr2 and vmbr3 existed with 10.99.0.1 and 10.98.0.1
- Recognize these IPs conflict with planned OPNSense VLAN gateways
- Clean up BEFORE starting VLAN configuration

**What actually happened:**
- Started VLAN configuration without checking existing bridges
- Only discovered the conflict during connectivity testing (3 hours later)
- Had to perform cleanup reactively under time pressure

**Impact:** Wasted 3 hours of work before discovering the core issue

---

### Mistake 3: Incomplete Awareness of Network Config File Structure

**When:** During cleanup attempt  
**Context:** Reviewing /etc/network/interfaces file  
**What should have happened:**
- Recognize the "source /etc/network/interfaces.d" directive
- Check for additional configuration files:
  ```bash
  ls -la /etc/network/interfaces.d/
  cat /etc/network/interfaces.d/*
  ```
- Identify vmbr2 and vmbr3 configuration files
- Delete those files before removing running bridges

**What actually happened:**
- Looked at /etc/network/interfaces and said "it's clean"
- Completely ignored the source directive
- Never checked /etc/network/interfaces.d/
- Only deleted bridges from running system, not configuration files

**Impact:** Set up the catastrophic failure that occurred on reboot

---

### Mistake 4: Rebooting Without Validation

**When:** After deleting running bridges  
**Context:** Cleanup appeared complete  
**What should have happened:**
- Test changes with "ifreload -a" first (non-destructive)
- If that worked, THEN reboot
- Have physical console access prepared before rebooting
- Document current working config as backup
- Create explicit rollback procedure

**What actually happened:**
- Went straight to reboot without testing
- No physical console access prepared
- No backup of working configuration
- No rollback plan

**Impact:** Lost all remote access with no recovery path

---

### Mistake 5: Not Having Physical Console Access Prepared

**When:** Throughout entire implementation  
**Context:** Making significant network changes  
**What should have happened:**
- Before starting VLAN work, ensure physical console access available
- For homelab: Connect keyboard/monitor OR set up remote console (iLO/IPMI)
- Test physical console access before making changes
- Document console access procedure

**What actually happened:**
- No physical console access available
- When network failed, had no way to access system
- Node completely inaccessible

**Impact:** Cannot recover without physically going to the machine

---

### Mistake 6: No Pre-Change Validation Checklist

**When:** Throughout implementation  
**Context:** Complex multi-step configuration  
**What should have happened:**

Create and follow a pre-change checklist:
- [ ] Current configuration documented
- [ ] Backup of current configuration created
- [ ] All existing bridges identified
- [ ] Configuration file locations verified
- [ ] Change plan documented step-by-step
- [ ] Rollback plan documented
- [ ] Testing plan documented
- [ ] Physical console access verified
- [ ] Cluster quorum verified before starting
- [ ] VM status documented before changes

**What actually happened:**
- No checklist
- Ad-hoc approach to changes
- Reactive problem-solving

**Impact:** Missed multiple opportunities to catch issues before they became critical

---

### Mistake 7: Not Testing in Non-Production First

**When:** Throughout implementation  
**Context:** This is a production homelab cluster  
**What should have happened:**
- Create a test VM or test node
- Test VLAN configuration in isolated environment
- Verify end-to-end connectivity works
- Document working procedure
- THEN apply to production

**What actually happened:**
- Tested directly on production cluster
- No isolated testing environment
- First attempt was on live infrastructure

**Impact:** Production impact when things went wrong

---

## Lessons Learned

### Technical Lessons

**1. Understand the Full Configuration Structure**

Proxmox network configuration is split across:
- /etc/network/interfaces - Main configuration
- /etc/network/interfaces.d/* - Modular additional configs
- Runtime state (ip link, ip addr) - What's actually running

Changes must be made to ALL relevant locations, not just one.

**Command to check full configuration:**
```bash
# Check main config
cat /etc/network/interfaces

# Check for additional configs
ls -la /etc/network/interfaces.d/
cat /etc/network/interfaces.d/*

# Check running state
ip addr show
ip link show
bridge link show
```

---

**2. Cleanup Requires Removing Both Runtime and Configuration**

To remove a network interface:
1. Remove from runtime: `ip link delete <interface>`
2. Remove from config files: Delete from /etc/network/interfaces AND /etc/network/interfaces.d/
3. Verify with: `grep -r <interface> /etc/network/`
4. Test reload: `ifreload -a`
5. Only THEN reboot

**Never delete runtime state without removing configuration files.**

---

**3. Test Network Changes Before Rebooting**

Proxmox provides `ifreload` command for testing:
```bash
# Test changes without reboot
ifreload -a

# If successful, networking reloads
# If failed, old config remains active

# Only after successful ifreload should you reboot
```

**Always test with ifreload before committing to a reboot.**

---

**4. Cluster Quorum is Fragile During Network Changes**

Changing a node's network configuration can temporarily break cluster communication. This is expected, but must be managed:

- Expect quorum loss during changes
- Have plan to quickly restore quorum
- Know how to update /etc/corosync/corosync.conf
- Understand quorum requirements (majority of nodes)

**Make network changes during maintenance windows.**

---

**5. Bridge Conflicts Prevent Routing**

If a Linux bridge has the same IP as a destination you're trying to reach, the host will respond locally instead of forwarding traffic.

**Example of conflict:**
- vmbr2 on host: 10.99.0.1/24
- VM wants to reach: 10.99.0.1 (expecting OPNSense)
- Host intercepts: Traffic goes to local vmbr2, never leaves host

**Always check for IP conflicts between host interfaces and gateway destinations.**

---

**6. VLAN-Aware Bridges Require Proper Configuration**

For Proxmox to pass VLAN-tagged traffic:
1. Bridge must be VLAN-aware: `bridge-vlan-aware yes`
2. VMs must be on that VLAN-aware bridge: `bridge=vmbr0`
3. VMs must have VLAN tag set: `tag=40`
4. Destination firewall must have VLAN interface configured

**All four layers must be correctly configured for VLAN traffic to work.**

---

### Process Lessons

**7. Always Have Physical Console Access**

For any network changes on infrastructure:
- Have keyboard/monitor available, OR
- Have IPMI/iLO/iDRAC configured and tested, OR
- Have serial console access configured

**Test console access BEFORE making changes, not after things break.**

---

**8. Create and Follow a Pre-Change Checklist**

Every significant infrastructure change needs:
- Current state documentation
- Configuration backups
- Step-by-step change plan
- Rollback plan
- Testing plan
- Validation criteria
- Required access methods documented

**Do not proceed without checklist completion.**

---

**9. Test in Non-Production First**

For complex changes like VLAN implementation:
- Create test environment (VM, spare node, or isolated cluster)
- Test entire procedure end-to-end
- Document what works
- Document what fails and how to fix
- THEN apply to production with confidence

**Never test complex changes on production first.**

---

**10. Clean Up Fully When Removing Services**

When deleting a VM or service:
- Remove the VM/service itself
- Remove ALL associated configurations
- Remove network bridges created for that service
- Remove firewall rules specific to that service
- Verify no remnants remain

**Incomplete cleanup creates technical debt that causes future failures.**

---

**11. Question Every Assumption**

When troubleshooting:
- "The config file is clean" - Did I check ALL config files?
- "The bridges are gone" - Are they gone from runtime AND config?
- "This will work after reboot" - Did I test with ifreload first?

**Verify, don't assume.**

---

**12. Have Explicit Rollback Plans**

Before making changes:
- Document current working state
- Create config file backups
- Write specific rollback steps
- Test rollback procedure if possible
- Know recovery options if rollback fails

**"We can always revert" is not a plan without documented steps.**

---

## Recovery Plan

### Immediate Actions Required

**1. Obtain Physical Console Access**

Connect keyboard and monitor to pve-gateway physical machine.

**2. Login at Physical Console**

Login as admin or root at text console.

**3. Check Current Network State**

```bash
# Check what interfaces exist
ip link show

# Check what has IP addresses
ip addr show

# Check if vmbr0 exists
ip link show vmbr0

# Check networking service status
systemctl status networking
```

**4. Check Configuration Files**

```bash
# List all config files
ls -la /etc/network/interfaces.d/

# Show contents
cat /etc/network/interfaces.d/*

# Look for vmbr2 or vmbr3
grep -r "vmbr2" /etc/network/
grep -r "vmbr3" /etc/network/
```

**5. Remove Legacy Configuration Files**

```bash
# If vmbr2/vmbr3 config files exist, remove them
cd /etc/network/interfaces.d/
rm vmbr2*
rm vmbr3*

# Verify they're gone
ls -la /etc/network/interfaces.d/
```

**6. Verify Main Network Config**

```bash
# Ensure /etc/network/interfaces only has vmbr0 and vmbr1
cat /etc/network/interfaces

# Should look like:
# auto vmbr0
# iface vmbr0 inet static
#     address 10.0.0.10/24
#     gateway 10.0.0.1
#     bridge-ports nic0
#     bridge-stp off
#     bridge-fd 0
#     bridge-vlan-aware yes
```

**7. Restart Networking**

```bash
# Attempt to bring up networking
systemctl restart networking

# Or use ifreload
ifreload -a

# Check if vmbr0 is up
ip addr show vmbr0

# Should show:
# vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.0.0.10/24 ...
```

**8. Test Network Connectivity**

```bash
# Test gateway
ping -c 3 10.0.0.1

# Test another node
ping -c 3 10.0.0.12

# Test internet
ping -c 3 8.8.8.8
```

**9. Restart SSH if Needed**

```bash
systemctl restart sshd
```

**10. Verify Cluster Rejoins**

```bash
# Check cluster status
pvecm status

# Should show:
# Quorate: Yes
# Nodes: 4
# All nodes with 10.0.0.x addresses
```

**11. If Networking Still Fails**

If networking service won't start even after cleaning config files:

```bash
# Check for syntax errors
cat /etc/network/interfaces

# Check for file permission issues
ls -la /etc/network/interfaces
ls -la /etc/network/interfaces.d/

# Check systemd networking logs
journalctl -u networking -n 50

# Try manual interface bring-up
ip link set nic0 up
ip link set vmbr0 up
ip addr add 10.0.0.10/24 dev vmbr0
ip route add default via 10.0.0.1

# Test connectivity
ping 10.0.0.1
```

**12. Last Resort: Revert to Known Good Config**

If nothing works, recreate /etc/network/interfaces from scratch:

```bash
# Backup broken config
cp /etc/network/interfaces /etc/network/interfaces.broken

# Create minimal working config
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
EOF

# Remove all files from interfaces.d
rm -f /etc/network/interfaces.d/*

# Restart networking
systemctl restart networking

# Verify
ip addr show vmbr0
ping 10.0.0.1
```

---

### Post-Recovery Actions

**1. Verify Full Cluster Health**

```bash
# On pve-gateway after recovery
pvecm status  # Should show quorate with 4 nodes

# On another node
ssh admin@10.0.0.12
pvecm status  # Should see pve-gateway back online
```

**2. Verify VM Status**

Check that all VMs on pve-gateway are intact:
```bash
qm list
```

**3. Document Recovery**

Record:
- What was found in /etc/network/interfaces.d/
- What files were removed
- What steps were needed to restore connectivity
- Any unexpected issues encountered

**4. Create Network Config Backup**

```bash
# Create backup directory
mkdir -p /root/network-backups/

# Backup current working config
cp /etc/network/interfaces /root/network-backups/interfaces.$(date +%Y%m%d)
cp -r /etc/network/interfaces.d/ /root/network-backups/interfaces.d.$(date +%Y%m%d)

# Document what's backed up
ls -la /root/network-backups/
```

---

### Resume VLAN Implementation (After Recovery)

**Only proceed after:**
- Network fully recovered
- Cluster healthy
- Physical console access remains available
- All steps documented and reviewed

**Modified VLAN Implementation Procedure:**

**1. Verify Clean State**

```bash
# Verify NO legacy bridges exist
ip addr show | grep vmbr

# Should only show vmbr0 and vmbr1
# NO vmbr2 or vmbr3

# Verify NO legacy config files
ls -la /etc/network/interfaces.d/
# Should be empty or only contain non-bridge configs

# Verify VLAN-aware is enabled
cat /sys/class/net/vmbr0/bridge/vlan_filtering
# Should show: 1
```

**2. Verify OPNSense VLAN Status**

OPNSense VLANs should still be configured from earlier:
- VLAN 40: 10.99.0.1/24
- VLAN 41: 10.98.0.1/24

Check Interfaces → Overview to confirm.

**3. Verify VM VLAN Tags**

VMs should still have VLAN tags from earlier:
```bash
qm config 300 | grep net0  # Should show tag=40
qm config 301 | grep net0  # Should show tag=40
qm config 302 | grep net0  # Should show tag=40
qm config 400 | grep net0  # Should show tag=41
```

**4. Test Connectivity Step-by-Step**

**Test 1: Proxmox host to OPNSense VLAN gateway**
```bash
ping -c 3 10.99.0.1
ping -c 3 10.98.0.1
```

Should SUCCEED (no bridge conflict now).

**Test 2: Start ONE VM and test**
```bash
qm start 300  # Kali VM
```

Wait 30 seconds for boot.

```bash
# From VM console
ping -c 3 10.99.0.1  # Gateway
ping -c 3 8.8.8.8    # Internet
ping -c 3 google.com # DNS
```

If all succeed, VLAN implementation is working.

**Test 3: Verify OPNSense sees VM**

In OPNSense:
- Interfaces → Diagnostics → ARP Table
- Look for 10.99.0.10

Should be present now.

**Test 4: Verify VLAN isolation**

Start VM 400 (malware VM on VLAN 41):
```bash
qm start 400
```

From VM console:
```cmd
ping 10.98.0.1   # Gateway - should work
ping 10.0.0.10   # Management - should work
ping 8.8.8.8     # Internet - should FAIL (isolated)
```

If isolation works as expected, VLAN implementation is complete and functional.

**5. Start Remaining VMs**

```bash
qm start 301  # metasploitable2
qm start 302  # DVWA
```

Test connectivity on each.

---

## Best Practices Moving Forward

### Network Change Procedure

For any future network configuration changes on Proxmox:

**Phase 1: Pre-Change (DO NOT SKIP)**

1. **Document Current State**
   ```bash
   # Save current config
   cp /etc/network/interfaces /root/network-backups/interfaces.$(date +%Y%m%d_%H%M)
   cp -r /etc/network/interfaces.d/ /root/network-backups/interfaces.d.$(date +%Y%m%d_%H%M)
   
   # Document current running state
   ip addr show > /root/network-backups/ip-addr.$(date +%Y%m%d_%H%M)
   ip route show > /root/network-backups/ip-route.$(date +%Y%m%d_%H%M)
   bridge link show > /root/network-backups/bridge-links.$(date +%Y%m%d_%H%M)
   ```

2. **Verify Physical Console Access**
   - Connect keyboard/monitor OR
   - Test IPMI/iLO access OR
   - Have someone on-site who can access console
   - TEST console access before proceeding

3. **Verify Cluster Health**
   ```bash
   pvecm status  # Quorate: Yes, all nodes visible
   ```

4. **Check for Existing Configurations**
   ```bash
   # Check all network configs
   cat /etc/network/interfaces
   ls -la /etc/network/interfaces.d/
   cat /etc/network/interfaces.d/*
   
   # Check running state
   ip addr show
   ip link show
   bridge link show
   
   # Look for anything unexpected
   ```

5. **Create Change Plan Document**
   - What will be changed
   - Why it's being changed
   - Step-by-step procedure
   - Expected outcome
   - Validation tests
   - Rollback procedure

---

**Phase 2: Making Changes**

1. **Make Changes to Config Files First**
   ```bash
   # Edit config files
   nano /etc/network/interfaces
   
   # OR add modular config
   nano /etc/network/interfaces.d/vlan-config
   ```

2. **Validate Syntax**
   ```bash
   # Check for obvious errors
   cat /etc/network/interfaces
   
   # Verify no conflicts
   grep -r "address" /etc/network/
   ```

3. **Test Without Reboot**
   ```bash
   # Test reload (does not persist through reboot)
   ifreload -a
   
   # If successful, interfaces reload
   # If failed, previous config remains active
   
   # Verify interfaces came up
   ip addr show
   ```

4. **Test Connectivity**
   ```bash
   ping 10.0.0.1      # Gateway
   ping 10.0.0.12     # Another node
   ping 8.8.8.8       # Internet
   ```

5. **If Tests Pass, Commit Change**
   ```bash
   # Only after successful ifreload and tests
   reboot
   ```

6. **If Tests Fail, Rollback**
   ```bash
   # Restore backup config
   cp /root/network-backups/interfaces.YYYYMMDD_HHMM /etc/network/interfaces
   cp -r /root/network-backups/interfaces.d.YYYYMMDD_HHMM /etc/network/interfaces.d/
   
   # Reload
   ifreload -a
   
   # Verify rollback successful
   ip addr show
   ```

---

**Phase 3: Post-Change Validation**

1. **Verify Network Interfaces**
   ```bash
   ip addr show
   ip link show
   ip route show
   ```

2. **Verify Cluster Communication**
   ```bash
   pvecm status
   # Should show quorate with all nodes
   ```

3. **Test SSH from External Host**
   ```bash
   ssh admin@10.0.0.10
   ```

4. **Test VM Connectivity**
   - Start test VM
   - Verify network works
   - Test gateway reachability
   - Test internet access

5. **Document What Was Done**
   - What changed
   - Why it changed
   - How to verify it's working
   - How to rollback if needed

---

### Bridge Removal Procedure

When removing a network bridge from Proxmox:

**1. Check What Uses the Bridge**
```bash
# Check which VMs use this bridge
qm config <vmid> | grep bridge

# Check for any containers
pct config <ctid> | grep bridge

# List all VMs
qm list

# Check each VM's network config
for vm in $(qm list | awk 'NR>1 {print $1}'); do
    echo "VM $vm:"
    qm config $vm | grep bridge
done
```

**2. Move VMs to Different Bridge**

Before removing bridge, move ALL VMs off it:
```bash
# For each VM using the bridge
qm set <vmid> --net0 virtio,bridge=vmbr0
```

**3. Remove Bridge from Running System**
```bash
# Bring down bridge
ip link set <bridge> down

# Remove bridge
ip link delete <bridge>

# Verify it's gone
ip link show <bridge>  # Should error
```

**4. Remove Bridge from Config Files**
```bash
# Check main config
grep -n "<bridge>" /etc/network/interfaces

# Remove bridge section from file
nano /etc/network/interfaces
# Delete the entire bridge stanza

# Check interfaces.d directory
grep -r "<bridge>" /etc/network/interfaces.d/
ls -la /etc/network/interfaces.d/

# Remove any files containing bridge config
rm /etc/network/interfaces.d/<bridge>*
```

**5. Verify Clean Removal**
```bash
# No mention in any config
grep -r "<bridge>" /etc/network/

# Should return nothing

# Test reload
ifreload -a

# Should succeed without errors
```

**6. Only Then Reboot**
```bash
reboot
```

**7. Post-Reboot Verification**
```bash
# Verify bridge is gone
ip link show <bridge>  # Should error

# Verify other bridges still work
ip addr show vmbr0     # Should show correct IP
```

---

### VLAN Implementation Procedure

For future VLAN implementations:

**Phase 1: Planning**

1. **Document Current Network**
   - All subnets in use
   - All gateways
   - All VLANs (if any exist)
   - All bridge configurations

2. **Design VLAN Architecture**
   - VLAN IDs (1-4094)
   - Subnets for each VLAN
   - Gateway IPs for each VLAN
   - Firewall rules between VLANs

3. **Check for Conflicts**
   - No IP overlap between VLANs
   - No conflicts with existing bridges
   - No conflicts with existing subnets

4. **Create Implementation Checklist**
   - Step-by-step procedure
   - Validation tests
   - Rollback procedure

---

**Phase 2: Implementation**

1. **Create VLANs on Firewall/Router First**
   - OPNSense/pfSense VLAN interfaces
   - Assign IP addresses
   - Configure firewall rules
   - Verify interfaces UP

2. **Enable VLAN-Aware on Proxmox Bridge**
   ```bash
   # Edit config
   nano /etc/network/interfaces
   
   # Add to bridge section
   bridge-vlan-aware yes
   
   # Test without reboot
   ifreload -a
   
   # Verify
   cat /sys/class/net/vmbr0/bridge/vlan_filtering
   # Should show: 1
   ```

3. **Configure ONE Test VM**
   ```bash
   # Set VLAN tag on test VM
   qm set <vmid> --net0 virtio,bridge=vmbr0,tag=<vlan_id>
   
   # Verify config
   qm config <vmid> | grep net0
   ```

4. **Test Connectivity**
   ```bash
   # Start test VM
   qm start <vmid>
   
   # From VM console
   ping <vlan_gateway>
   ping 8.8.8.8
   ```

5. **If Test Succeeds, Configure Other VMs**
   - Apply VLAN tags to remaining VMs
   - Test each one
   - Verify isolation between VLANs

6. **Document Working Configuration**
   - VLAN IDs and subnets
   - VM assignments
   - Firewall rules
   - Validation procedure

---

### Recovery Access Preparation

**Before making ANY network changes:**

1. **Set Up Console Access**
   - Physical: Connect keyboard/monitor before changes
   - IPMI/iLO: Configure and test remote console
   - Serial console: Configure if available
   - Document console access procedure

2. **Test Console Access**
   - Actually login via console
   - Verify you can access system
   - Verify you can edit files
   - Document any password/authentication requirements

3. **Create Recovery USB**
   - Bootable Linux live USB
   - Can be used to mount and edit configs if system won't boot
   - Test booting from USB

4. **Document Recovery Procedures**
   - How to access console
   - How to edit network configs from console
   - How to restart networking service
   - Emergency contact information

---

### Configuration Management

**Implement version control for configs:**

```bash
# Create git repo for configs
cd /etc
git init
git add network/interfaces
git add network/interfaces.d/
git commit -m "Baseline network config"

# Before making changes
git status
git diff

# After changes
git add network/
git commit -m "Added VLAN configuration"

# To rollback
git log  # Find commit to restore
git checkout <commit> network/
```

**Or use simpler versioned backups:**

```bash
# Create backup script
cat > /usr/local/bin/backup-network-config.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/root/network-backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

cp /etc/network/interfaces $BACKUP_DIR/interfaces.$DATE
cp -r /etc/network/interfaces.d/ $BACKUP_DIR/interfaces.d.$DATE
ip addr show > $BACKUP_DIR/ip-addr.$DATE
ip route show > $BACKUP_DIR/ip-route.$DATE

echo "Backup created: $BACKUP_DIR/*.$DATE"
EOF

chmod +x /usr/local/bin/backup-network-config.sh

# Run before any changes
/usr/local/bin/backup-network-config.sh
```

---

### Testing Procedure

**Create standard test script:**

```bash
cat > /usr/local/bin/test-network.sh << 'EOF'
#!/bin/bash

echo "Testing network connectivity..."

# Test gateway
echo -n "Gateway (10.0.0.1): "
if ping -c 1 -W 2 10.0.0.1 &>/dev/null; then
    echo "OK"
else
    echo "FAIL"
    exit 1
fi

# Test another node
echo -n "Peer node (10.0.0.12): "
if ping -c 1 -W 2 10.0.0.12 &>/dev/null; then
    echo "OK"
else
    echo "FAIL"
    exit 1
fi

# Test internet
echo -n "Internet (8.8.8.8): "
if ping -c 1 -W 2 8.8.8.8 &>/dev/null; then
    echo "OK"
else
    echo "FAIL"
    exit 1
fi

# Test DNS
echo -n "DNS (google.com): "
if ping -c 1 -W 2 google.com &>/dev/null; then
    echo "OK"
else
    echo "FAIL"
    exit 1
fi

echo "All tests passed"
exit 0
EOF

chmod +x /usr/local/bin/test-network.sh

# Run after any changes
/usr/local/bin/test-network.sh
```

---

## Summary and Action Items

### What Went Wrong

1. **Legacy pfSense bridges (vmbr2, vmbr3) not cleaned up yesterday**
2. **Started VLAN work without checking for existing bridge conflicts**
3. **Discovered conflict only after 3 hours of troubleshooting**
4. **During cleanup, only removed bridges from runtime, not config files**
5. **Did not check /etc/network/interfaces.d/ for additional configs**
6. **Rebooted without testing with ifreload**
7. **No physical console access available**
8. **System failed to boot with network, leaving node completely inaccessible**

### Current State

- pve-gateway: OFFLINE (no network)
- Remaining cluster: OPERATIONAL (3 of 4 nodes)
- VLANs: CONFIGURED on OPNSense but untested
- Recovery: Requires physical console access

### Immediate Actions

1. **Obtain physical console access to pve-gateway**
2. **Remove vmbr2/vmbr3 config files from /etc/network/interfaces.d/**
3. **Restart networking service**
4. **Verify network comes up**
5. **Test cluster rejoins**
6. **Only then resume VLAN testing**

### Long-Term Improvements

1. **Implement console access solution (IPMI or permanent KVM)**
2. **Create configuration backup automation**
3. **Implement change management checklist**
4. **Create network testing script**
5. **Document all procedures**
6. **Test changes in non-production first**
7. **Never reboot without testing with ifreload**
8. **Always verify ALL config file locations before changes**

### Prevention Strategy

**The checklist that would have prevented this:**

- [ ] Physical console access available and tested
- [ ] Current configuration backed up
- [ ] ALL config files identified (interfaces + interfaces.d)
- [ ] Existing bridges documented
- [ ] No IP conflicts identified
- [ ] Change tested with ifreload before reboot
- [ ] Rollback procedure documented
- [ ] Recovery procedure documented

**IF this checklist had been followed, the failure would not have occurred.**

---

## Conclusion

This incident demonstrates the critical importance of:
- Understanding the full extent of system configuration
- Thorough pre-change validation
- Testing before committing changes
- Having multiple access methods available
- Following systematic procedures instead of ad-hoc approaches

The technical mistake (not checking /etc/network/interfaces.d/) was small, but the process failures (no checklist, no testing, no console access) turned it into a catastrophic failure.

The path forward requires:
1. Physical access to recover the system
2. Completion of the configuration cleanup
3. Implementation of proper change management procedures
4. Systematic approach to VLAN implementation

With these lessons learned and procedures in place, future network changes can be made safely and successfully.

---

**Document Status:** COMPLETE  
**Recovery Status:** PENDING (requires physical console)  
**VLAN Implementation Status:** INCOMPLETE (on hold pending recovery)  
**Next Steps:** Physical console access to restore pve-gateway network connectivity
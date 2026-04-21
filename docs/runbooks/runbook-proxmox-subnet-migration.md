# Proxmox Homelab Network Migration Report
## From 192.168.100.x to 10.0.0.x Subnet

**Date:** January 9, 2026  
**Duration:** ~4 hours (Day 4 of OPNSense deployment saga)  
**Outcome:** ✓ Successful  
**Difficulty:** Medium-High (complicated by prior configuration attempts)

---

## Executive Summary

Successfully migrated a 4-node Proxmox cluster and all infrastructure VMs from 192.168.100.0/24 subnet to 10.0.0.0/24 subnet. This migration was necessary to resolve a fundamental routing conflict where OPNSense WAN and LAN interfaces were both configured on the same subnet (192.168.100.x), which violated basic routing principles and made proper firewall deployment impossible.

The migration cleared the path for proper OPNSense deployment with correct network architecture.

---

## Initial State (Before Migration)

### Network Topology
```
Heimdall (Pi 5 @ 192.168.100.1) - WiFi bridge to home network
    ↓ patch panel
Unmanaged Switch
    ↓
├─ pve-gateway (192.168.100.2)
├─ pve-services (192.168.100.3)
├─ pve-env2 (192.168.100.4)
├─ pve-env1 (192.168.100.7)
└─ Workstation (DHCP 192.168.100.x)
```

### Infrastructure Inventory

**Proxmox Hosts:**
- pve-gateway: 192.168.100.2 (HP Mini - designated OPNSense hardware)
- pve-services: 192.168.100.3 (HP Mini - Docker host)
- pve-env2: 192.168.100.4 (HP Mini - development)
- pve-env1: 192.168.100.7 (HP Mini - PKI/CA infrastructure)

**Virtual Machines:**
- VM 100 (pfsense): 192.168.100.5 - Old firewall VM, to be retired
- VM 200 (services-host): 192.168.100.6 - Docker Host 1
- VM 300 (Kali-attack): 10.99.0.10 - Security lab (isolated)
- VM 301 (metasploitable2): 10.99.0.20 - Security lab (isolated)
- VM 302 (DVWA): 10.99.0.30 - Security lab (isolated)
- VM 400 (malware-win11): 10.98.0.10 - Malware analysis (air-gapped)
- VM 401 (ubuntu): 192.168.100.77 - Development machine
- VM 500 (pve-ca-root): 192.168.100.60 - Root CA
- VM 501 (pve-ca-intermediate): 192.168.100.61 - Intermediate CA
- VM 700 (VM 700): 192.168.100.66 - Docker Host 2

### The Problem

**Previous 3 days** were spent attempting to configure OPNSense with both WAN and LAN on 192.168.100.x subnet:
- WAN: 192.168.100.250 or .254 (connecting to heimdall at .1)
- LAN: 192.168.100.251 or .253 (connecting to Proxmox nodes at .2, .3, .4, .7)

**This violated fundamental routing principles:**
1. **Routing table ambiguity** - Kernel doesn't know which interface owns the subnet
2. **ARP ownership conflict** - Both interfaces claim authority over same broadcast domain
3. **Reverse-path filtering failure** - Replies exit wrong interface, packets dropped

**Result:** Asymmetric routing, broken connectivity, repeated configuration failures despite multiple XML edits, static routes, virtual IPs, and VLAN attempts.

---

## Migration Plan (Final Correct Approach)

### New IP Scheme

**Management Network (10.0.0.0/24):**
- 10.0.0.1 - OPNSense LAN gateway (to be configured)
- 10.0.0.10 - pve-gateway
- 10.0.0.11 - pve-services
- 10.0.0.12 - pve-env1
- 10.0.0.13 - pve-env2
- 10.0.0.20 - pve-ca-root (VM 500)
- 10.0.0.21 - pve-ca-intermediate (VM 501)
- 10.0.0.30 - services-host (VM 200)
- 10.0.0.31 - VM 700 (Docker Host 2)
- 10.0.0.40 - ubuntu (VM 401)
- 10.0.0.100-254 - DHCP pool (future)

**WAN Network (192.168.100.0/24):**
- 192.168.100.1 - Heimdall (Pi 5)
- 192.168.100.250 - OPNSense WAN (to be configured)

**Security Lab Networks (VLANs - to be configured on OPNSense):**
- 192.168.40.0/24 - VLAN 40 (Security Lab 1)
- 192.168.41.0/24 - VLAN 41 (Security Lab 2)

### Migration Strategy

1. **Physically bypass OPNSense** - Connect heimdall directly to switch
2. **Renumber Proxmox hosts** one at a time (node + VMs)
3. **Update Proxmox cluster configuration** (corosync.conf)
4. **Fix SSH hardening** on secured nodes
5. **Verify connectivity** before moving to next node
6. **Cleanup** - Power off security labs, retire pfSense VM
7. **Reconnect OPNSense** and configure properly

---

## Step-by-Step Migration Process

### Phase 0: Physical Network Bypass

**Actions Taken:**
1. Unplugged OPNSense WAN cable from heimdall
2. Unplugged OPNSense LAN cable from switch
3. Connected heimdall's Ethernet directly to switch

**Result:**
```
Heimdall (192.168.100.1)
    ↓
Switch → All Proxmox nodes + Workstation
```

Network reverted to pre-OPNSense state, allowing direct access to all infrastructure.

**Workstation Configuration:**
- IP: 192.168.100.199/24
- Gateway: 192.168.100.1
- DNS: 192.168.100.1

**Verification:**
```bash
ping 192.168.100.1  # heimdall
ping 192.168.100.2  # pve-gateway
ping 192.168.100.3  # pve-services
```

---

### Phase 1: Migrate pve-env1 (First Node)

**Rationale:** Started with pve-env1 as it's the least critical node (PKI/CA infrastructure is important but VMs can be offline briefly).

#### Step 1.1: Change Proxmox Host IP

**Access:** https://192.168.100.7:8006

**Actions:**
1. System → Network
2. Edit vmbr0 interface
3. Changed IPv4/CIDR: 192.168.100.7/24 → 10.0.0.12/24
4. Changed Gateway: 192.168.100.1 → 10.0.0.1
5. Applied Configuration
6. **Lost connection (expected)**

#### Step 1.2: Change Workstation IP

**Windows Network Settings:**
- IP: 10.0.0.199/24
- Subnet: 255.255.255.0
- Gateway: 10.0.0.1 (doesn't exist yet)
- DNS: 192.168.100.1 (still on old subnet for now)

**Verification:**
```cmd
ping 10.0.0.12  # ✓ Success
```

**Access:** https://10.0.0.12:8006

#### Step 1.3: Discovered Quorum Loss

**Problem:** Could not start VMs on pve-env1.

**Error:** "no quorum"

**Diagnosis:**
```bash
pvecm status
# Showed: Nodes: 1, Quorate: No
# Only pve-env1 visible, other nodes offline
```

**Root Cause:** pve-env1 at 10.0.0.12 could not communicate with other cluster members still at 192.168.100.x (different subnet, no routing).

**Decision:** Rapid-fire migrate remaining nodes to restore quorum.

---

### Phase 2: Rapid Cluster Migration (Restore Quorum)

#### Step 2.1: Migrate pve-env2

**Workstation IP Change:**
- Changed back to: 192.168.100.199/24 to access pve-env2

**Access:** https://192.168.100.4:8006

**Actions:**
1. System → Network → vmbr0 → Edit
2. Changed to: 10.0.0.13/24, gateway 10.0.0.1
3. Applied Configuration

**Workstation IP Change:**
- Changed to: 10.0.0.199/24

**Verification:**
```cmd
ping 10.0.0.13  # ✓ Success
```

#### Step 2.2: Migrate pve-services

**Workstation IP Change:**
- Changed back to: 192.168.100.199/24

**Access:** https://192.168.100.3:8006

**Actions:**
1. System → Network → vmbr0 → Edit
2. Changed to: 10.0.0.11/24, gateway 10.0.0.1
3. Applied Configuration

**Workstation IP Change:**
- Changed to: 10.0.0.199/24

**Verification:**
```cmd
ping 10.0.0.11  # ✓ Success
```

#### Step 2.3: Migrate pve-gateway

**Workstation IP Change:**
- Changed back to: 192.168.100.199/24

**Access:** https://192.168.100.2:8006

**Actions:**
1. Powered off security lab VMs first:
   - VM 300 (Kali-attack)
   - VM 301 (metasploitable2)
   - VM 302 (DVWA)
   - VM 400 (malware-win11)
2. System → Network → vmbr0 → Edit
3. Changed to: 10.0.0.10/24, gateway 10.0.0.1
4. Applied Configuration

**Workstation IP Change:**
- Changed to: 10.0.0.199/24

**Verification:**
```cmd
ping 10.0.0.10  # ✓ Success
```

---

### Phase 3: Fix Proxmox Cluster Configuration

#### Step 3.1: Check Cluster Status

**Problem:** All nodes online, but cluster still not communicating.

**From any node:**
```bash
pvecm status
```

**Output:**
```
Quorum information
------------------
Nodes:            1
Quorate:          No
Expected votes:   4
Total votes:      1

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 192.168.100.2 (local)
```

**Root Cause:** Corosync cluster configuration still had old 192.168.100.x IPs.

#### Step 3.2: Update Corosync Configuration

**Accessed pve-env1:**
```bash
ssh admin@10.0.0.12
sudo nano /etc/corosync/corosync.conf
```

**Original nodelist:**
```
nodelist {
  node {
    name: pve-gateway
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 192.168.100.2
  }
  node {
    name: pve-services
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 192.168.100.3
  }
  node {
    name: pve-env2
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 192.168.100.4
  }
  node {
    name: pve-env1
    nodeid: 4
    quorum_votes: 1
    ring0_addr: 192.168.100.7
  }
}
```

**Updated nodelist:**
```
nodelist {
  node {
    name: pve-gateway
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 10.0.0.10
  }
  node {
    name: pve-services
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 10.0.0.11
  }
  node {
    name: pve-env2
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 10.0.0.13
  }
  node {
    name: pve-env1
    nodeid: 4
    quorum_votes: 1
    ring0_addr: 10.0.0.12
  }
}
```

#### Step 3.3: Distribute Configuration to All Nodes

**SSH hardening issue:** Could not `scp` directly to pve-gateway and pve-services due to SSH key restrictions.

**Solution:** Manually edited corosync.conf on each node via SSH.

**On each node:**
```bash
ssh admin@10.0.0.XX
sudo nano /etc/corosync/corosync.conf
# Made same changes
sudo systemctl restart corosync
```

#### Step 3.4: Verify Cluster

**From any node:**
```bash
pvecm status
```

**Output:**
```
Cluster information
-------------------
Name:             homelab
Config Version:   6
Transport:        knet
Secure auth:      on

Quorum information
------------------
Quorate:          Yes
Nodes:            4
Expected votes:   4
Total votes:      4

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.0.0.10
0x00000002          1 10.0.0.11
0x00000003          1 10.0.0.13
0x00000004          1 10.0.0.12 (local)
```

**✓ Success:** Cluster quorum restored!

---

### Phase 4: Migrate Virtual Machines

With cluster healthy, proceeded to migrate VMs on each node.

#### VM Migration Approach

**For each VM:**
1. Access Proxmox host WebUI
2. Open VM Console
3. Login to guest OS
4. Edit network configuration (Netplan for Ubuntu VMs)
5. Apply changes
6. Verify connectivity

**Challenge:** VMs use Netplan (Ubuntu) for network configuration, requires editing YAML files inside guest OS.

---

#### Phase 4.1: pve-env1 VMs

##### VM 401 (ubuntu) - Development Machine

**Current IP:** 192.168.100.77 (DHCP)  
**New IP:** 10.0.0.40/24

**Access:** pve-env1 WebUI → VM 401 → Console

**Actions:**
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Original configuration:**
```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      dhcp4: true
```

**New configuration:**
```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      addresses:
        - 10.0.0.40/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
        addresses:
          - 192.168.100.1
```

**Applied:**
```bash
sudo netplan apply
ip addr show enp6s18
ping 10.0.0.12  # pve-env1 host
```

**Note:** Gateway 10.0.0.1 doesn't exist yet (will be OPNSense), but VM can reach local subnet.

**✓ Success**

---

##### VM 500 (pve-ca-root) - Root Certificate Authority

**Current IP:** 192.168.100.60  
**New IP:** 10.0.0.20/24

**Access:** pve-env1 WebUI → VM 500 → Console

**Discovery:** IP was already configured as 10.0.0.20, but gateway was wrong.

**Actions:**
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Changed gateway:**
```yaml
via: "192.168.100.1"  →  via: "10.0.0.1"
```

**Applied:**
```bash
sudo netplan apply
ping 10.0.0.12
```

**✓ Success**

---

##### VM 501 (pve-ca-intermediate) - Intermediate Certificate Authority

**Current IP:** 192.168.100.61  
**New IP:** 10.0.0.21/24

**Access:** pve-env1 WebUI → VM 501 → Console

**Configuration:** Already correct (10.0.0.21/24, gateway 10.0.0.12 temporary)

**Verification:**
```bash
ping 10.0.0.12
```

**✓ Success**

---

#### Phase 4.2: pve-env2 VMs

##### VM 700 (Docker Host 2)

**Current IP:** 192.168.100.66  
**New IP:** 10.0.0.31/24

**Access:** pve-env2 WebUI → VM 700 → Console

**Configuration:** Already correct (10.0.0.31/24, gateway 10.0.0.13 temporary)

**Note:** `ping` not installed on this VM, but netplan config verified as correct.

**✓ Success**

---

#### Phase 4.3: pve-services VMs

##### Problem: Cannot Access pve-services WebUI

**Error:** "Connection error 595: No route to host"

**Diagnosis:**
- Can ping 10.0.0.11 from workstation ✓
- Cannot access WebUI ✗
- Cannot SSH as admin@10.0.0.11 ✗

**Root Cause:** Proxmox web console trying to connect to OLD IP (192.168.100.3) instead of new IP (10.0.0.11).

**Attempted Fixes:**
1. Updated /etc/hosts on all nodes
2. Restarted pve-cluster, pvedaemon, pveproxy services
3. Cleared SSH known_hosts

**Result:** Still could not access via web console.

**Solution:** Physical console access required.

---

##### VM 200 (services-host) - Docker Host 1

**Current IP:** 192.168.100.6  
**New IP:** 10.0.0.30/24

**Access:** Physical console on pve-services

**Actions:**

1. **Fixed pve-services /etc/hosts:**
```bash
sudo nano /etc/hosts
```

Changed to:
```
127.0.0.1 localhost.localdomain localhost
10.0.0.11 pve-services.local pve-services
10.0.0.10 pve-gateway.local pve-gateway
10.0.0.12 pve-env1.local pve-env1
10.0.0.13 pve-env2.local pve-env2
```

2. **Fixed SSH configuration:**
```bash
sudo nano /etc/ssh/sshd_config
# Added: AllowUsers admin@10.0.0.0/24
sudo systemctl restart sshd
sudo systemctl restart pve-cluster
```

3. **Accessed VM 200 Console via Proxmox WebUI** (now working)

4. **Updated VM 200 network:**
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**Configuration:** Already correct (10.0.0.30/24, gateway 10.0.0.1)

**Applied:**
```bash
sudo netplan apply
```

**✓ Success**

---

#### Phase 4.4: pve-gateway Cleanup

##### Problem: SSH Access to pve-gateway

**Initial attempt:** SSH failed due to SSH hardening restrictions.

**Workaround:** Had root terminal access via different method.

##### Actions:

1. **Fixed /etc/hosts on pve-gateway:**
```bash
sudo nano /etc/hosts
```

Changed to:
```
127.0.0.1 localhost.localdomain localhost
10.0.0.10 pve-gateway.local pve-gateway
10.0.0.11 pve-services.local pve-services
10.0.0.12 pve-env1.local pve-env1
10.0.0.13 pve-env2.local pve-env2
```

2. **Fixed SSH configuration:**
```bash
sudo nano /etc/ssh/sshd_config
# Added: AllowUsers admin@10.0.0.0/24
sudo systemctl restart sshd
```

3. **Powered off security lab VMs:**
```bash
qm stop 300  # Kali-attack
qm stop 301  # metasploitable2
qm stop 302  # DVWA
qm stop 400  # malware-win11
```

4. **Deleted pfSense VM:**
```bash
qm destroy 100
```

**✓ Success**

---

### Phase 5: Final Cleanup and Verification

#### Step 5.1: Update All /etc/hosts Files

**Ensured all 4 nodes had correct /etc/hosts:**
```
127.0.0.1 localhost.localdomain localhost
10.0.0.10 pve-gateway.local pve-gateway
10.0.0.11 pve-services.local pve-services
10.0.0.12 pve-env1.local pve-env1
10.0.0.13 pve-env2.local pve-env2
```

#### Step 5.2: Restart Services on All Nodes

**On each node:**
```bash
rm /root/.ssh/known_hosts
systemctl restart pve-cluster
systemctl restart pvedaemon
systemctl restart pveproxy
```

#### Step 5.3: Final Verification

**Cluster status:**
```bash
pvecm status
# Quorate: Yes
# Nodes: 4
# All nodes showing 10.0.0.x IPs
```

**SSH connectivity:**
```bash
ssh admin@10.0.0.10  # ✓ pve-gateway
ssh admin@10.0.0.11  # ✓ pve-services
ssh admin@10.0.0.12  # ✓ pve-env1
ssh admin@10.0.0.13  # ✓ pve-env2
```

**VM connectivity:**
```bash
ping 10.0.0.20  # ✓ pve-ca-root
ping 10.0.0.21  # ✓ pve-ca-intermediate
ping 10.0.0.30  # ✓ services-host
ping 10.0.0.31  # ✓ VM 700
ping 10.0.0.40  # ✓ ubuntu
```

**✓ All systems operational**

---

## Final State (After Migration)

### Network Topology
```
Heimdall (192.168.100.1) - WiFi bridge to home network
    ↓
Switch (temporary bypass of OPNSense)
    ↓
├─ pve-gateway (10.0.0.10) - Ready for OPNSense deployment
├─ pve-services (10.0.0.11)
├─ pve-env1 (10.0.0.12)
├─ pve-env2 (10.0.0.13)
└─ Workstation (10.0.0.199)
```

**Next step:** Reconnect OPNSense and configure properly with WAN on 192.168.100.x and LAN on 10.0.0.x (different subnets).

### Migrated Assets

**Proxmox Hosts:**
- ✓ pve-gateway: 192.168.100.2 → 10.0.0.10
- ✓ pve-services: 192.168.100.3 → 10.0.0.11
- ✓ pve-env1: 192.168.100.7 → 10.0.0.12
- ✓ pve-env2: 192.168.100.4 → 10.0.0.13

**Virtual Machines:**
- ✓ VM 200: 192.168.100.6 → 10.0.0.30 (services-host/Docker Host 1)
- ✓ VM 401: 192.168.100.77 → 10.0.0.40 (ubuntu development)
- ✓ VM 500: 192.168.100.60 → 10.0.0.20 (pve-ca-root)
- ✓ VM 501: 192.168.100.61 → 10.0.0.21 (pve-ca-intermediate)
- ✓ VM 700: 192.168.100.66 → 10.0.0.31 (Docker Host 2)

**Powered Off (for later VLAN migration):**
- VM 300: Kali-attack (currently 10.99.0.10)
- VM 301: metasploitable2 (currently 10.99.0.20)
- VM 302: DVWA (currently 10.99.0.30)
- VM 400: malware-win11 (currently 10.98.0.10)

**Deleted:**
- VM 100: pfSense (no longer needed)

---

## What Went Wrong

### 1. Three Days of Attempted OPNSense Configuration (Days 1-3)

**Problem:** Repeatedly configured OPNSense with WAN and LAN both on 192.168.100.x subnet.

**Why It Failed:**
- Violated fundamental routing invariant: A router cannot route between two interfaces in the same L3 subnet
- Created routing table ambiguity (kernel doesn't know which interface owns the subnet)
- Created ARP ownership conflict (both interfaces claim authority)
- Caused reverse-path filtering failures (replies exit wrong interface, dropped)

**Attempted Workarounds (All Failed):**
- XML config editing
- Static host routes
- Virtual IP addresses
- VLAN 100 on LAN side (still same subnet conflict)
- ARP cache manipulation
- Multiple gateway changes

**Result:** Wasted 3 days going in circles, repeated configuration attempts, constant IP address changes on workstation, mounting frustration.

### 2. Quorum Loss During Migration

**Problem:** After changing pve-env1 to 10.0.0.12, cluster lost quorum and VMs could not be started.

**Root Cause:** Other nodes still on 192.168.100.x could not communicate with pve-env1 on different subnet.

**Why This Happened:** Did not anticipate cluster communication requirements during migration.

**Impact:** Had to rapidly migrate all remaining nodes before VMs could be accessed, adding time pressure.

### 3. SSH Hardening Complications

**Problem:** Could not SSH to pve-services and pve-gateway with normal methods after IP change.

**Root Cause:** Previous SSH hardening configured to only allow connections from 192.168.100.0/24 subnet.

**Why This Happened:** Forgot about SSH hardening restrictions when planning migration.

**Impact:** Required physical console access to pve-services, added significant time and inconvenience.

### 4. /etc/hosts Not Updated Systematically

**Problem:** Proxmox web console trying to connect to old IPs even after network changes applied.

**Root Cause:** /etc/hosts on each node still contained old 192.168.100.x addresses for other nodes.

**Why This Happened:** Did not have systematic checklist of all files needing updates during IP migration.

**Impact:** Could not access web consoles, required manual /etc/hosts updates on all 4 nodes.

### 5. Corosync Configuration Delays

**Problem:** Cluster quorum not restored even after all nodes migrated.

**Root Cause:** /etc/corosync/corosync.conf still had old ring0_addr IPs.

**Why This Happened:** Not familiar enough with Proxmox cluster internals, did not anticipate this requirement.

**Impact:** Additional troubleshooting time, had to manually edit config on all nodes.

---

## What We Could Have Done Better

### 1. Recognize the Fundamental Problem Immediately (Day 1)

**What Happened:** Spent 3 days trying to make same-subnet WAN/LAN work.

**What Should Have Happened:** On Day 1, immediately recognize:
- "WAN and LAN on same subnet = impossible"
- "This requires renumbering Proxmox nodes"
- Stop attempting workarounds, accept the renumbering requirement

**Lesson:** When violating fundamental networking principles, no amount of creative configuration will fix it. Accept the architectural requirement and plan accordingly.

**Time Saved:** 2-3 days

---

### 2. Create Complete Pre-Migration Checklist

**What We Should Have Done:**

**Phase 0: Planning**
- [ ] Document all IPs (hosts and VMs)
- [ ] Choose new IP scheme
- [ ] Identify SSH hardening on nodes
- [ ] Identify cluster-dependent services
- [ ] Verify VM network configuration methods
- [ ] Create rollback plan

**Phase 1: Preparation**
- [ ] Backup cluster configuration (`pvecm config`)
- [ ] Backup VM configurations
- [ ] Take VM snapshots
- [ ] Document current cluster status
- [ ] Prepare physical console access if needed

**Phase 2: Migration Checklist (Per Node)**
- [ ] Change Proxmox host network interface
- [ ] Update /etc/hosts
- [ ] Update /etc/corosync/corosync.conf
- [ ] Update SSH restrictions (if applicable)
- [ ] Restart cluster services
- [ ] Verify node reachable
- [ ] Change VMs on that node
- [ ] Verify VMs reachable
- [ ] Test inter-node connectivity

**Phase 3: Post-Migration**
- [ ] Verify cluster quorum
- [ ] Test SSH to all nodes
- [ ] Test VM connectivity
- [ ] Clear SSH known_hosts
- [ ] Restart services cluster-wide
- [ ] Document new configuration

**Impact:** Would have prevented SSH hardening surprise, /etc/hosts delays, corosync config oversight.

---

### 3. Migrate All Hosts Before Touching VMs

**What Happened:** Started migrating VMs on pve-env1, hit quorum loss.

**Better Approach:**

**Step 1:** Migrate ALL Proxmox hosts rapidly
- Accept workstation IP will change frequently
- Get all hosts on new subnet ASAP
- Update corosync.conf immediately
- Restore cluster quorum

**Step 2:** Then systematically migrate VMs node by node
- Cluster is healthy
- No time pressure
- Can take VMs offline safely

**Benefit:** No quorum loss, no "can't start VMs" panic, more organized process.

---

### 4. Fix SSH Hardening BEFORE Migration

**What Happened:** Discovered SSH hardening blocking access DURING migration, required physical console.

**Better Approach:**

**Before migrating pve-services and pve-gateway:**
1. SSH in while still on old subnet
2. Update SSH config preemptively:
   ```bash
   # Add new subnet BEFORE migrating
   AllowUsers admin@10.0.0.0/24 admin@192.168.100.0/24
   ```
3. Restart SSH
4. Then migrate the node
5. After migration, remove old subnet from SSH config

**Benefit:** No physical console access needed, cleaner process.

---

### 5. Use Configuration Management

**What We Did:** Manually edited /etc/hosts, /etc/corosync/corosync.conf on each node individually.

**Better Approach:** Use Ansible, Puppet, or even a simple bash script:

```bash
#!/bin/bash
# update-cluster-ips.sh

NEW_HOSTS="127.0.0.1 localhost.localdomain localhost
10.0.0.10 pve-gateway.local pve-gateway
10.0.0.11 pve-services.local pve-services
10.0.0.12 pve-env1.local pve-env1
10.0.0.13 pve-env2.local pve-env2"

NODES="10.0.0.10 10.0.0.11 10.0.0.12 10.0.0.13"

for node in $NODES; do
  echo "Updating $node..."
  ssh admin@$node "echo '$NEW_HOSTS' | sudo tee /etc/hosts"
  ssh admin@$node "sudo systemctl restart pve-cluster"
done
```

**Benefit:** Faster, less error-prone, repeatable, auditable.

---

### 6. Have Physical Console Access Prepared

**What Happened:** Needed physical console for pve-services, had to walk to rack, connect keyboard/monitor.

**Better Approach:**
- Set up iLO/IPMI on all nodes beforehand
- OR have KVM switch permanently connected
- OR use USB serial adapters for emergency console

**Benefit:** Remote console access without physical presence.

---

### 7. Test Migration Process on Single Node First

**What We Could Have Done:**

**Test Run (Non-Production):**
1. Spin up test Proxmox cluster (or single node)
2. Simulate IP change
3. Document all issues encountered
4. Refine process
5. Then execute on production

**Benefit:** Identify SSH hardening, quorum, corosync issues in safe environment.

---

### 8. Communicate Network Architecture Earlier

**What Happened:** User discovered the same-subnet problem through trial and error over 3 days.

**Better Approach:**
On Day 1, provide clear diagram:

```
❌ WRONG (Will Not Work):
Heimdall (192.168.100.1)
    ↓
OPNSense WAN (192.168.100.254)  ← Same subnet
    ↓
OPNSense LAN (192.168.100.253)  ← Same subnet
    ↓
Proxmox (192.168.100.x)

✓ CORRECT (Will Work):
Heimdall (192.168.100.1)
    ↓
OPNSense WAN (192.168.100.250)  ← Different subnet
    ↓
OPNSense LAN (10.0.0.1)         ← Different subnet
    ↓
Proxmox (10.0.0.x)
```

Explain: "This requires renumbering Proxmox nodes. Here's how..."

**Benefit:** Set correct expectations, avoid 3 days of failed attempts.

---

### 9. Document Temporary vs. Permanent Gateways

**Confusion During Migration:**
- Some VMs had gateway: 10.0.0.1 (future OPNSense, doesn't exist yet)
- Some had gateway: 10.0.0.XX (their Proxmox host, temporary)
- Unclear which was correct

**Better Approach:**

**During migration, explicitly state:**
- "Set gateway to 10.0.0.1 (final OPNSense gateway, won't work until OPNSense configured)"
- "VMs can communicate locally but no internet until OPNSense deployed"
- "This is expected and correct"

**Benefit:** Reduce confusion about non-working gateways.

---

### 10. Use Infrastructure-as-Code for VMs

**What We Did:** Manually edited netplan config files on each VM via console.

**Better Approach:**
- Use cloud-init for VM network configuration
- Store configs in version control
- Apply programmatically via Proxmox API
- Or use Terraform for VM provisioning

**Benefit:** Faster VM network changes, less manual console work, reproducible.

---

## Key Lessons Learned

### Technical Lessons

1. **Routing Fundamentals Are Non-Negotiable**
   - A router MUST have different subnets on WAN and LAN
   - No configuration workaround can violate this principle
   - Accept architectural requirements, don't fight them

2. **Proxmox Cluster Has Multiple Configuration Files**
   - `/etc/network/interfaces` - Network config
   - `/etc/hosts` - Name resolution
   - `/etc/corosync/corosync.conf` - Cluster membership
   - `/etc/ssh/sshd_config` - SSH access
   - ALL must be updated for IP migration

3. **Quorum Matters**
   - Proxmox cluster needs majority of nodes to function
   - IP changes break cluster communication
   - Plan migration to minimize quorum loss time

4. **SSH Hardening Has Consequences**
   - IP-based restrictions require updates during migrations
   - Always have backup console access method
   - Consider adding new IPs BEFORE removing old ones

5. **Network Configuration Varies by Guest OS**
   - Ubuntu: netplan (YAML-based)
   - Debian: /etc/network/interfaces
   - CentOS: /etc/sysconfig/network-scripts
   - Containers: May be set by host
   - Must know which method each VM uses

### Process Lessons

6. **Plan Complete Migration Before Starting**
   - Document all assets
   - Identify dependencies
   - Create checklist
   - Prepare rollback plan
   - Don't start until plan is complete

7. **Migrate Infrastructure Before Applications**
   - Get hosts stable first
   - Then migrate workloads
   - Minimizes complexity and troubleshooting surface

8. **Have Physical Access Prepared**
   - Remote management (iLO/IPMI) is critical
   - OR have KVM console ready
   - Don't assume SSH will always work

9. **Test in Non-Production First**
   - Complex migrations deserve rehearsal
   - Identifies issues safely
   - Refines process

10. **Automate Repetitive Tasks**
    - Manual editing across 4+ nodes is error-prone
    - Scripts or configuration management save time
    - Creates audit trail

### Communication Lessons

11. **Set Correct Expectations Early**
    - Be direct about what's possible vs. impossible
    - Don't let user attempt impossible workarounds
    - Explain "why" behind architectural requirements

12. **Use Visual Diagrams**
    - Network topology diagrams clarify architecture
    - Before/after comparisons show what changes
    - Worth 1000 words of explanation

13. **Document As You Go**
    - Don't rely on memory
    - Record each step taken
    - Capture error messages and solutions
    - Enables learning from mistakes

---

## Time Analysis

### Actual Time Spent

**Days 1-3 (Previous Attempts):** ~12-15 hours
- Repeated OPNSense configuration attempts
- XML editing
- Static route troubleshooting
- VLAN configuration attempts
- Multiple IP address changes on workstation
- Growing frustration

**Day 4 (This Migration):** ~4 hours
- Phase 0: Physical bypass (15 min)
- Phase 1: pve-env1 migration (30 min)
- Phase 2: Rapid cluster migration (30 min)
- Phase 3: Corosync config updates (45 min)
- Phase 4: VM migrations (1.5 hours)
- Phase 5: Final cleanup (30 min)

**Total:** ~16-19 hours

### Optimal Time Estimate (With Better Planning)

**With proper planning and checklist:**
- Phase 0: Physical bypass (10 min)
- Phase 1: All hosts migration (1 hour)
- Phase 2: Cluster config update (30 min)
- Phase 3: VM migrations (1 hour)
- Phase 4: Verification (15 min)

**Estimated Total:** ~2.5-3 hours

**Time That Could Have Been Saved:** 13-16 hours

---

## Recommendations for Future Migrations

### Immediate Actions

1. **Document This Migration**
   - Save this report
   - Create runbook for future IP changes
   - Include all config files that need updates

2. **Set Up Remote Management**
   - Enable iLO/IPMI on all nodes
   - OR install USB serial adapters
   - Test remote console access

3. **Create Backup Procedures**
   - Automate cluster config backups
   - Store VM configs in version control
   - Regular snapshots of critical VMs

4. **Update SSH Hardening Strategy**
   - Allow management subnet (10.0.0.0/24)
   - Keep SSH restrictions reasonable
   - Document all hardening configurations

### Before Next Migration

5. **Create Migration Toolkit**
   - Ansible playbooks for common changes
   - Scripts for cluster updates
   - Pre-migration checklist template
   - Post-migration verification script

6. **Test Disaster Recovery**
   - Practice full cluster rebuild
   - Verify backups actually work
   - Document recovery procedures

7. **Improve Monitoring**
   - Set up monitoring for cluster health
   - Alert on quorum loss
   - Track node connectivity

### Long-Term Improvements

8. **Infrastructure as Code**
   - Terraform for VM provisioning
   - Ansible for configuration management
   - Git for version control
   - CI/CD for testing changes

9. **Network Documentation**
   - Maintain network topology diagrams
   - Document all subnets and VLANs
   - Keep IP allocation spreadsheet
   - Record all firewall rules

10. **Knowledge Transfer**
    - Document lessons learned
    - Create troubleshooting guides
    - Build internal wiki
    - Share with other homelab operators

---

## Conclusion

This migration successfully resolved a fundamental network architecture problem that prevented proper OPNSense deployment. While the process took longer than optimal due to lack of planning and unforeseen complications (SSH hardening, cluster quorum, corosync configuration), it provided valuable lessons in Proxmox cluster management, network architecture, and migration planning.

The key takeaway: **When fundamental networking principles are violated (same subnet on WAN and LAN), no amount of creative configuration will fix it. Accept the architectural requirement (separate subnets) and plan the migration properly.**

With proper planning, checklists, and automation, a similar migration in the future should take 2-3 hours instead of 16-19 hours.

### Next Steps

1. Configure OPNSense with correct architecture:
   - WAN: 192.168.100.250/24 (to heimdall)
   - LAN: 10.0.0.1/24 (rack gateway)
   
2. Create VLANs for security labs:
   - VLAN 40: 192.168.40.0/24
   - VLAN 41: 192.168.41.0/24
   
3. Migrate security lab VMs to VLANs

4. Test end-to-end connectivity

5. Document final network architecture

---

## Appendix: Quick Reference

### New IP Allocations

**Management Network (10.0.0.0/24):**
- 10.0.0.1 - OPNSense LAN (gateway)
- 10.0.0.10 - pve-gateway
- 10.0.0.11 - pve-services
- 10.0.0.12 - pve-env1
- 10.0.0.13 - pve-env2
- 10.0.0.20 - pve-ca-root (VM 500)
- 10.0.0.21 - pve-ca-intermediate (VM 501)
- 10.0.0.30 - services-host (VM 200)
- 10.0.0.31 - VM 700 (Docker Host 2)
- 10.0.0.40 - ubuntu (VM 401)
- 10.0.0.100-254 - DHCP pool

**WAN Network (192.168.100.0/24):**
- 192.168.100.1 - Heimdall
- 192.168.100.250 - OPNSense WAN

### Critical Config Files

**Proxmox Cluster:**
- `/etc/network/interfaces` - Network configuration
- `/etc/hosts` - Name resolution
- `/etc/corosync/corosync.conf` - Cluster membership (ring0_addr)
- `/etc/ssh/sshd_config` - SSH access restrictions

**Ubuntu VMs:**
- `/etc/netplan/*.yaml` - Network configuration

### Useful Commands

```bash
# Cluster status
pvecm status

# List VMs
qm list

# Start/stop VM
qm start <vmid>
qm stop <vmid>

# VM console
qm terminal <vmid>

# Restart cluster services
systemctl restart pve-cluster
systemctl restart corosync
systemctl restart pvedaemon
systemctl restart pveproxy

# Network config on Ubuntu VMs
sudo netplan apply
ip addr show
ip route show
```

---

**Report Generated:** January 9, 2026  
**Migration Duration:** ~4 hours (Day 4), ~16-19 hours total (Days 1-4)  
**Status:** ✓ Successful  
**Next Phase:** OPNSense Configuration
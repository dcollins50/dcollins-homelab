# Security Lab VLAN Connectivity Troubleshooting Report

**Date:** January 24, 2026  
**System:** pve-gateway Proxmox host  
**Issue:** VMs on VLAN 40/41 completely unreachable from all network locations  
**Time to Resolution:** ~2 hours of active troubleshooting

---

## Executive Summary

Security lab VMs (Kali, metasploitable2, DVWA on VLAN 40; malware-win11 on VLAN 41) running on pve-gateway were completely unreachable despite correct VLAN tagging on VMs, proper OPNSense VLAN configuration, and working switch configuration. Root cause: Proxmox VLAN-aware bridge was not configured to pass VLAN 40/41 traffic on the physical interface (nic0), despite VMs being correctly tagged. 

**Root Cause:** Missing `bridge-vids` configuration on pve-gateway's vmbr0, preventing VLAN 40/41 traffic from traversing the physical interface even though the bridge was set to `bridge-vlan-aware yes`.

---

## Initial Symptoms

1. **Complete connectivity failure:**
   - Cannot ping security lab VMs from workstation (10.0.0.199)
   - Cannot ping security lab VMs from OPNSense (10.0.0.1)
   - Cannot ping security lab VMs from any Proxmox host
   - VMs are running and appear healthy in Proxmox interface

2. **Contrast with working systems:**
   - Other VLANs (20, 30) work perfectly
   - Can ping services-host on pve-services @ 10.0.20.30
   - Can ping VM 700 on pve-SOC @ 10.0.20.31
   - Can ping all Proxmox hosts from all locations
   - **Can ping OPNSense VLAN 40 gateway (10.99.0.1) from workstation**

3. **Expected vs actual:**
   - VMs should respond at 10.99.0.10 (Kali), 10.99.0.20 (metasploitable2), 10.99.0.30 (DVWA), 10.98.0.10 (malware VM)
   - All pings time out with 100% packet loss

---

## Network Topology Context

```
Workstation (10.0.0.199)
    ↓
OPNSense LAN (10.0.0.1) --- VLAN 1 (Management)
    |
    +--- VLAN 40 interface (10.99.0.1) --- Security Lab 1
    +--- VLAN 41 interface (10.98.0.1) --- Security Lab 2
    ↓
TP-Link Managed Switch (Port 5)
    ↓
pve-gateway nic0 (Port 1-4, one of these)
    ↓
vmbr0 (bridge-vlan-aware)
    ↓
VM tap interfaces (tag=40, tag=41)
    ↓
Security Lab VMs
```

**Key insight that should have been obvious:** Workstation could reach OPNSense's VLAN 40 gateway (10.99.0.1) but not VMs (10.99.0.x). This meant OPNSense VLAN interfaces were working, but traffic wasn't reaching the VMs.

---

## Troubleshooting Steps Performed

### Phase 1: Configuration Verification (15 minutes)

**Goal:** Verify all configurations are correct before deep troubleshooting.

#### 1.1 VM Network Configuration
```bash
# On pve-gateway
qm config 300  # Kali
qm config 301  # metasploitable2
qm config 302  # DVWA
qm config 400  # malware-win11
```

**Result:** All VMs correctly configured:
- VM 300, 301, 302: `bridge=vmbr0,tag=40` ✓
- VM 400: `bridge=vmbr0,tag=41` ✓

#### 1.2 Proxmox Bridge Configuration
```bash
cat /etc/network/interfaces
```

**Result:** vmbr0 configuration appeared correct:
```
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

**CRITICAL OVERSIGHT:** Did not verify `bridge vlan show` output at this stage. This would have immediately revealed the issue.

#### 1.3 OPNSense VLAN Interfaces
**Interfaces → Other Types → VLAN:**
- VLAN 40: Parent re0, Tag 40, Description "VLAN40_SecurityLab1" ✓
- VLAN 41: Parent re0, Tag 41, Description "VLAN41_SecurityLab2" ✓

**Interfaces → Assignments:**
- OPT1 assigned to re0_vlan40 with IP 10.99.0.1/24, Status: UP ✓
- OPT2 assigned to re0_vlan41 with IP 10.98.0.1/24, Status: UP ✓

**OPNSense Shell:**
```bash
ifconfig | grep -A 5 vlan
ping -c 3 10.99.0.1  # Success
ping -c 3 10.98.0.1  # Success
```

**Result:** OPNSense can reach its own VLAN gateways ✓

#### 1.4 TP-Link Switch VLAN Configuration
**VLAN → 802.1Q VLAN:**
- VLAN 40 exists ✓
- VLAN 41 exists ✓
- Member Ports: 1-5 (all Proxmox hosts and OPNSense) ✓
- Tagged Ports: 1-5 ✓

**Result:** Switch properly configured to pass VLAN 40/41 as tagged on all relevant ports ✓

#### 1.5 Firewall Rules
**OPNSense → Firewall → Rules:**
- VLAN40_SecurityLab1: Rules exist allowing traffic ✓
- VLAN41_SecurityLab2: Rules exist allowing traffic ✓
- LAN: Rules exist allowing LAN → VLAN40 and VLAN41 ✓

**Result:** Firewall not blocking traffic ✓

### Phase 2: Layer 2/3 Diagnostics (30 minutes)

#### 2.1 VM-to-VM Communication Test
**From Kali console (VM 300):**
```bash
ping -c 3 10.99.0.30  # DVWA
```

**Result:** Success ✓ - VMs on same VLAN/host can communicate

**Significance:** This proves:
- VLAN tagging working within Proxmox
- vmbr0 bridge functioning for intra-host traffic
- VM network interfaces working
- Problem is external connectivity

#### 2.2 Individual VM Health Check
**VM 301 (metasploitable2) had no IPv4 address** - this was a separate issue but not the root cause. Kali and DVWA had correct IPs and still couldn't be reached.

### Phase 3: Packet Flow Analysis (45 minutes)

**This phase consumed the most time but ultimately led to the solution.**

#### 3.1 Inbound Traffic to VMs - Physical Interface
```bash
# On pve-gateway
sudo tcpdump -i nic0 -nn vlan 40
```

**While OPNSense pings 10.99.0.10:**
```
12:28:34.601434 ARP, Request who-has 10.99.0.10 tell 10.99.0.1, length 42
12:28:35.636345 ARP, Request who-has 10.99.0.10 tell 10.99.0.1, length 42
```

**Result:** ARP requests FROM OPNSense ARE arriving at pve-gateway's nic0 ✓

#### 3.2 Inbound Traffic - Bridge
```bash
sudo tcpdump -i vmbr0 -nn vlan 40
```

**Result:** Same ARP requests visible on the bridge ✓

**Significance:** Switch and physical interface working, bridge receiving traffic

#### 3.3 Inbound Traffic - VM Interface
**From Kali console:**
```bash
sudo tcpdump -i eth0 -nn
```

**Result:** Kali IS receiving ARP requests from 10.99.0.1 ✓

**Significance:** Complete inbound path working - OPNSense → Switch → nic0 → vmbr0 → VM

#### 3.4 Outbound Traffic from VM - Bridge Level
```bash
# On pve-gateway
sudo tcpdump -i vmbr0 -nn src 10.99.0.10
```

**While Kali runs arping:**
```bash
# In Kali
sudo arping -I eth0 10.99.0.1
```

**Result:** vmbr0 shows ARP requests FROM Kali asking for 10.99.0.1 ✓

**Significance:** VM can send, bridge receives VM's packets

#### 3.5 Outbound Traffic - Physical Interface (CRITICAL TEST)
```bash
# On pve-gateway
sudo tcpdump -i nic0 -nn -e vlan 40 and src 10.99.0.10
```

**While Kali runs arping:**

**Result:** **NO PACKETS CAPTURED** ✗

**THIS WAS THE BREAKTHROUGH:** Packets from Kali reach vmbr0 but NEVER leave nic0. The bridge is not forwarding VLAN 40 traffic to the physical interface.

#### 3.6 OPNSense Receive Verification
**OPNSense → Interfaces → Diagnostics → Packet Capture:**
- Interface: VLAN40_SecurityLab1
- Protocol: ARP

**While Kali arpings:**

**Result:** **NOTHING CAPTURED** ✗

**Confirmation:** VLAN 40 traffic is not reaching OPNSense at all.

---

## Root Cause Identification

### The Diagnostic Command That Revealed Everything

```bash
# On pve-gateway
/sbin/bridge vlan show
```

**Output:**
```
port              vlan-id
nic0              1 PVID Egress Untagged
vmbr0             1 PVID Egress Untagged
vmbr1             1 PVID Egress Untagged
tap300i0          40 PVID Egress Untagged
tap301i0          40 PVID Egress Untagged
tap302i0          40 PVID Egress Untagged
tap400i0          41 PVID Egress Untagged
```

**Analysis:**
- **tap interfaces (VMs):** VLAN 40 and 41 configured ✓
- **nic0 (physical interface):** VLAN 1 ONLY ✗

**Root Cause Identified:** The physical interface (nic0) was not configured to carry VLAN 40 or 41 traffic. When VMs send VLAN-tagged packets, the bridge cannot forward them out nic0 because nic0 doesn't have those VLANs in its allowed list.

### Why Other VLANs Worked

This explains why VLAN 20 (Services) worked on other Proxmox hosts but VLAN 40/41 didn't work on pve-gateway. Other hosts either:
1. Had `bridge-vids` explicitly configured in `/etc/network/interfaces`
2. Had VLANs manually added to their physical interfaces
3. Were configured differently during initial setup

---

## Solution Implementation

### Immediate Fix (Temporary)
```bash
# Add VLAN 40 to physical interface
sudo /sbin/bridge vlan add vid 40 dev nic0

# Add VLAN 41 to physical interface  
sudo /sbin/bridge vlan add vid 41 dev nic0

# Verify
/sbin/bridge vlan show
```

**Result after fix:**
```
nic0              1 PVID Egress Untagged
                  40
                  41
```

**Testing:**
```bash
# From Kali
ping -c 3 10.99.0.1
# Success!

# From workstation
ping 10.99.0.10
# Success!

# From OPNSense
ping -c 3 10.99.0.10
# Success!
```

### Permanent Fix
```bash
sudo nano /etc/network/interfaces
```

**Modified vmbr0 configuration:**
```
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

**Key addition:** `bridge-vids 2-4094` allows all VLANs (2-4094) to traverse the physical interface.

---

## Lessons Learned

### What Should Have Been Done Differently

1. **Run `bridge vlan show` immediately** when:
   - VMs can communicate with each other ✓
   - But cannot reach gateway ✗
   - Other VLANs work on other hosts ✓
   
   This single command would have revealed the issue in under 5 minutes.

2. **Systematic packet flow tracing from the start:**
   - Should have started at the physical interface (nic0) to see if tagged packets were leaving
   - Then worked backward to find where packets stop flowing
   - Instead, we verified configurations first (which were "correct" but incomplete)

3. **Compare working vs non-working systems:**
   - Should have checked `bridge vlan show` on pve-services (where VLAN 20 works) vs pve-gateway
   - Would have revealed the difference in bridge VLAN configuration

4. **Document bridge VLAN requirements explicitly:**
   - Setting `bridge-vlan-aware yes` alone is NOT sufficient
   - Must also configure which VLANs are allowed on trunk ports (physical interfaces)
   - This is done via `bridge-vids` in config or `bridge vlan add` commands

### What Was Overlooked During Initial Setup

1. **Incomplete VLAN-aware bridge configuration:**
   - Assumed `bridge-vlan-aware yes` would automatically pass all VLANs
   - Reality: You must explicitly allow VLANs on each bridge port (especially trunk ports like nic0)
   - Should have added `bridge-vids 2-4094` to the initial vmbr0 configuration

2. **Inconsistent configuration across Proxmox hosts:**
   - Other hosts (pve-services, pve-env1, pve-SOC) apparently had proper bridge-vids configuration
   - pve-gateway was configured differently, possibly because it was set up at a different time
   - Should have standardized all Proxmox host network configs with a template

3. **Testing methodology:**
   - After configuring VLANs on OPNSense and setting VM tags, should have immediately tested connectivity
   - Instead, assumed "if configs look right, it will work"
   - Failed fast principle: Test immediately after each change

4. **Documentation of Proxmox VLAN requirements:**
   - Should have documented that Proxmox VLAN-aware bridges require TWO things:
     - `bridge-vlan-aware yes` (enables VLAN filtering)
     - `bridge-vids X-Y` (specifies which VLANs are allowed)
   - Or explicit `bridge vlan add` commands for each required VLAN

### Frustrations Experienced (Honest Assessment)

1. **Repetitive verification cycles:**
   - Spent significant time re-checking configurations that had already been confirmed correct
   - This was frustrating but ultimately necessary to rule out simple configuration errors
   - In hindsight, should have moved to packet-level diagnostics faster

2. **Assumption that "config looks right" means "it will work":**
   - The bridge configuration DID look correct at the `/etc/network/interfaces` level
   - But `bridge-vlan-aware yes` without `bridge-vids` is incomplete
   - This subtle distinction consumed the most troubleshooting time

3. **Working systems creating false confidence:**
   - Since VLAN 20 worked perfectly on other hosts, assumed the VLAN infrastructure was correct
   - Didn't consider that pve-gateway might be configured differently
   - Created tunnel vision focusing on OPNSense/firewall/switch instead of Proxmox bridge

4. **Tool availability:**
   - `bridge` command not being in PATH initially wasted time
   - Should have checked `/sbin/bridge` immediately or installed `iproute2` at start

---

## Correct Setup Procedure (For Future Reference)

### When Configuring VLANs on a Proxmox Host

1. **Enable VLAN-aware bridge AND specify allowed VLANs:**
   ```
   auto vmbr0
   iface vmbr0 inet static
       address X.X.X.X/24
       gateway X.X.X.X
       bridge-ports nicX
       bridge-stp off
       bridge-fd 0
       bridge-vlan-aware yes
       bridge-vids 2-4094    # <- CRITICAL: Don't forget this
   ```

2. **Verify bridge VLAN configuration immediately:**
   ```bash
   /sbin/bridge vlan show
   ```
   
   **Expected output for trunk port:**
   ```
   nicX              1 PVID Egress Untagged
                     2-4094
   ```

3. **Test connectivity for each VLAN before moving on:**
   - Create a test VM on the VLAN
   - Verify it can ping its gateway
   - Verify you can ping the VM from other networks
   - If ANY test fails, troubleshoot immediately

4. **Use consistent configuration across all hosts:**
   - Template the network configuration
   - Apply same bridge-vids settings to all Proxmox hosts with VLANs
   - Document any host-specific deviations

### Diagnostic Checklist for Future VLAN Issues

**When VMs on a specific VLAN can't be reached:**

1. ☐ **Verify Layer 1/2 basics:**
   ```bash
   # Are VMs actually running?
   qm list
   
   # Do VMs have correct IPs?
   # Console into VM, run: ip addr show
   ```

2. ☐ **Check VM VLAN tagging:**
   ```bash
   qm config <VMID> | grep net
   # Should show: bridge=vmbr0,tag=<VLAN_ID>
   ```

3. ☐ **Verify bridge VLAN configuration (CRITICAL):**
   ```bash
   /sbin/bridge vlan show
   # Physical interface MUST include the VLAN in question
   ```

4. ☐ **Check inbound packet flow:**
   ```bash
   # On Proxmox host
   tcpdump -i <physical_nic> -nn vlan <VLAN_ID>
   # Try to ping VM from gateway
   # Should see ARP/ICMP requests arriving
   ```

5. ☐ **Check outbound packet flow:**
   ```bash
   # On Proxmox host
   tcpdump -i <physical_nic> -nn src <VM_IP>
   # Ping from inside VM
   # Should see packets leaving
   ```

6. ☐ **Compare with working configuration:**
   ```bash
   # If other VLANs work on other hosts
   # Compare bridge vlan show output between hosts
   ```

---

## Technical Notes

### Understanding Proxmox VLAN-Aware Bridges

**Key Concept:** A VLAN-aware bridge in Proxmox acts as an 802.1Q VLAN switch. It can:
- Accept VLAN-tagged traffic from VMs (via tap interfaces)
- Forward traffic between VMs on the same VLAN without leaving the host
- Forward VLAN-tagged traffic to/from physical interfaces

**However:** The bridge must be explicitly told which VLANs are allowed on each port:
- **VM tap interfaces:** Configured automatically based on VM's `tag=X` setting
- **Physical interfaces:** Must be configured manually via `bridge-vids` or `bridge vlan add`

**Default behavior without bridge-vids:**
- Physical interface only carries VLAN 1 (untagged/PVID)
- VM tap interfaces with VLAN tags work for intra-host communication
- But VLAN-tagged traffic cannot leave/enter the host via the physical interface

### Why `bridge-vlan-aware yes` Isn't Enough

Setting `bridge-vlan-aware yes` enables VLAN filtering on the bridge, but it doesn't configure which VLANs are allowed. Think of it as:
- `bridge-vlan-aware yes` = "Turn on VLAN filtering mode"
- `bridge-vids X-Y` = "Allow these specific VLANs"

Without `bridge-vids`, the physical interface defaults to VLAN 1 only.

### Alternative Configuration Methods

**Method 1: Allow all VLANs (recommended for homelab):**
```
bridge-vids 2-4094
```

**Method 2: Allow specific VLANs only:**
```
bridge-vids 20 30 40 41 50 60
```

**Method 3: Runtime addition (temporary):**
```bash
bridge vlan add vid 40 dev nic0
bridge vlan add vid 41 dev nic0
```

---

## Conclusion

**Total troubleshooting time:** ~2 hours active debugging

**Time that could have been saved:** ~1.5 hours if `bridge vlan show` had been run immediately upon isolating the issue to pve-gateway

**Core lesson:** When using Proxmox VLAN-aware bridges, always verify `bridge vlan show` output matches your intended configuration. The `/etc/network/interfaces` file can look "correct" while the runtime bridge VLAN configuration is incomplete.

**Setup checklist moving forward:**
1. Always add `bridge-vids 2-4094` when enabling `bridge-vlan-aware yes`
2. Always run `bridge vlan show` after configuration changes
3. Test connectivity immediately after each VLAN is configured
4. Document and standardize network configurations across all hosts

**Silver lining:** This troubleshooting exercise provided deep understanding of Proxmox bridge VLAN mechanics, packet flow analysis with tcpdump, and systematic network troubleshooting methodology. Skills gained will significantly accelerate future debugging.
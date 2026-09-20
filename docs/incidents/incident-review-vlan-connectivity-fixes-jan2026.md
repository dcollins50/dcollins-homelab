# VLAN Connectivity Troubleshooting - January 24, 2026

**Systems Fixed:** pve-gateway (VLAN 40/41), pve-env1 (VLAN 30)  
**Time to Resolution:** ~3 hours total

---

## Issue #1: Security Lab VMs Unreachable (VLAN 40/41 on pve-gateway)

### Symptoms
- VMs on VLAN 40/41 completely unreachable from all locations
- Could ping OPNSense VLAN gateway (10.99.0.1, 10.98.0.1) but not VMs
- VMs could ping each other on same VLAN/host
- Kali (10.99.0.10), metasploitable2 (10.99.0.20), DVWA (10.99.0.30), malware-win11 (10.98.0.10)

### Root Cause
Proxmox bridge on pve-gateway not configured to pass VLAN 40/41 traffic on physical interface (nic0).

**The diagnostic command that revealed it:**
```bash
/sbin/bridge vlan show
```

Output showed:
- tap interfaces (VMs): VLAN 40/41 ✓
- nic0 (physical): VLAN 1 only ✗

### Fix

**Immediate (temporary):**
```bash
sudo /sbin/bridge vlan add vid 40 dev nic0
sudo /sbin/bridge vlan add vid 41 dev nic0
```

**Permanent:**
```bash
sudo nano /etc/network/interfaces
```

Modified vmbr0:
```
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.10/24
    gateway 10.0.0.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094        # <- Added this line
```

### Verification
```bash
# From Kali
ping -c 3 10.99.0.1          # Success
ping -c 3 10.99.0.30         # Success (DVWA)

# From workstation
ping 10.99.0.10              # Success (Kali)

# From OPNSense
ping -c 3 10.99.0.10         # Success
```

---

## Issue #2: VM 401 (Ubuntu) Can't Reach Gateway (VLAN 30 on pve-env1)

### Symptoms
- VM 401 (10.0.30.40) could ping workstation (10.0.0.199) ✓
- VM 401 could ping Management VLAN gateway (10.0.0.1) ✓
- VM 401 could NOT ping its own VLAN 30 gateway (10.0.30.1) ✗
- OPNSense could ping VM 401 ✓

### Diagnosis Process

**Step 1: Verify bridge configuration**
```bash
/sbin/bridge vlan show
```
Result: Bridge correctly configured with VLANs 2-4094 on nic0 ✓

**Step 2: Check VM IP configuration**
```bash
ip addr show
ip route show
```
Result: VM correctly configured with 10.0.30.40/24, gateway 10.0.30.1 ✓

**Step 3: Packet capture on pve-env1**
```bash
sudo tcpdump -i nic0 -nn -e vlan 30 and host 10.0.30.40
```
Result: Packets leaving nic0 with VLAN 30 tags ✓

**Step 4: OPNSense packet capture**
Interface: DEV, Protocol: ICMP  
Result: **OPNSense receiving pings but NOT responding** ✗

**Step 5: Test OPNSense → VM**
```bash
ping -S 10.0.30.1 10.0.30.40
```
Result: Works ✓

**Conclusion:** Firewall rules blocking VM → Gateway traffic

### Root Cause
Missing firewall rules on OPNSense DEV interface. Existing rule only allowed DEV → LAN, but not:
- Traffic to gateway itself
- DNS queries to Heimdall Pi-hole
- General internet access

### Fix

**On OPNSense: Firewall → Rules → DEV**

**Rule 1: Allow traffic to gateway**
- Action: Pass
- Interface: DEV
- Protocol: any
- Source: DEV net
- Destination: This Firewall
- Description: "Allow DEV to gateway"

**Rule 2: Allow DNS to Heimdall**
- Action: Pass
- Interface: DEV
- Protocol: TCP/UDP
- Source: DEV net
- Destination: Single host → 192.168.100.1
- Destination Port: 53
- Description: "Allow DEV to Heimdall DNS"

**Rule 3: Allow internet access**
- Action: Pass
- Interface: DEV
- Protocol: any
- Source: DEV net
- Destination: any
- Description: "Allow DEV to internet"

### Verification
```bash
# From VM 401
ping -c 3 10.0.30.1          # Gateway - Success
nslookup google.com          # DNS - Success
ping -c 3 google.com         # Internet - Success
apt update                   # Package updates - Success
```

---

## Key Lessons

### 1. Proxmox Bridge VLAN Configuration
Setting `bridge-vlan-aware yes` is NOT enough. You must also specify which VLANs are allowed:
- **Wrong:** Just `bridge-vlan-aware yes`
- **Correct:** `bridge-vlan-aware yes` + `bridge-vids 2-4094`

### 2. Diagnostic Commands
**Always run these when troubleshooting VLAN issues:**
```bash
/sbin/bridge vlan show              # Shows actual VLAN config on bridge ports
tcpdump -i <interface> vlan <id>    # Verify VLAN-tagged packets
```

### 3. OPNSense Firewall Rules
Default behavior: **Block all traffic unless explicitly allowed**

Required rules for a functional VLAN interface:
1. Allow traffic to gateway/firewall itself
2. Allow DNS (if using external DNS like Pi-hole)
3. Allow internet access (if needed)
4. Allow inter-VLAN traffic (if needed)

### 4. Systematic Troubleshooting
When VMs can't reach gateway:
1. Check bridge VLAN config (`bridge vlan show`)
2. Check VM IP/gateway config
3. Capture packets at each layer (tap → bridge → nic)
4. Check if packets reach firewall (OPNSense packet capture)
5. Check firewall rules

### 5. Test Bidirectional Connectivity
If A can ping B but B can't ping A, likely causes:
- Firewall rules (most common)
- Routing issues
- ARP problems

---

## Current VLAN Status

### Working VLANs
- ✅ **VLAN 1 (Management):** 10.0.0.0/24 - Proxmox hosts, CA VMs
- ✅ **VLAN 20 (Services):** 10.0.20.0/24 - Docker hosts (VM 200, VM 700)
- ✅ **VLAN 30 (Services2):** 10.0.30.0/24 - VM 401 (ubuntu)
- ✅ **VLAN 40 (Security Lab 1):** 10.99.0.0/24 - Kali, metasploitable2, DVWA
- ✅ **VLAN 41 (Security Lab 2):** 10.98.0.0/24 - malware-win11

### Untested VLANs
- ⚠️ **VLAN 50 (DMZ):** 10.0.50.0/24 - Future use
- ⚠️ **VLAN 60 (Storage):** 10.0.60.0/24 - Future use

---

## Configuration Checklist for Future VLANs

When adding a new VLAN to a Proxmox host:

**On Proxmox Host:**
- [ ] Verify `/etc/network/interfaces` has `bridge-vids 2-4094`
- [ ] Run `bridge vlan show` to confirm physical interface has VLAN
- [ ] Tag VM with correct VLAN in Proxmox GUI
- [ ] Test VM → bridge → physical interface with tcpdump

**On OPNSense:**
- [ ] Create VLAN interface (Interfaces → Other Types → VLAN)
- [ ] Assign VLAN to OPT interface
- [ ] Configure IP address on interface
- [ ] Add firewall rule: Allow VLAN → This Firewall
- [ ] Add firewall rule: Allow VLAN → DNS (if needed)
- [ ] Add firewall rule: Allow VLAN → Internet (if needed)
- [ ] Test with packet capture on OPNSense

**On TP-Link Switch:**
- [ ] Verify VLAN exists in 802.1Q VLAN table
- [ ] Verify trunk ports (1-5) are tagged members
- [ ] Verify PVID is correct (usually 1 for trunks)

---

## Quick Reference: Common Issues

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| VMs can talk to each other but not gateway | Bridge not passing VLAN on physical interface | Add `bridge-vids 2-4094` |
| Gateway can ping VM but VM can't ping gateway | Firewall rules missing | Add rules on VLAN interface |
| Packets leave Proxmox but don't reach OPNSense | Switch VLAN config | Check port memberships |
| DNS doesn't work | Wrong DNS server or no rule for DNS traffic | Fix DNS config + add firewall rule |

---

## Files Modified Today

**pve-gateway:**
- `/etc/network/interfaces` - Added `bridge-vids 2-4094` to vmbr0

**OPNSense:**
- Firewall rules on DEV interface (3 new rules)

---

**Documentation Date:** January 24, 2026  
**Issues Resolved:** 2  
**Systems Affected:** pve-gateway, pve-env1, OPNSense  
**All VLANs Tested:** Operational
# Standard Operating Procedure: VLAN Implementation with OPNSense and Managed Switch

**Document Version:** 2.0
**Last Updated:** 2026-04-21
**Author:** Daniel Collins
**Target Environment:** Homelab / Enterprise Network Segmentation

---

## Executive Summary

This SOP provides a complete, ordered procedure for implementing VLAN-based network segmentation using OPNSense firewall and a managed switch. The procedure is designed to be executed linearly without backtracking or reconfiguration. All common failure modes and pitfalls have been identified and accounted for in the ordering of steps.

**Estimated Time to Complete:** 2-3 hours for experienced administrators, 4-6 hours for first-time implementation.

**Critical Success Factors:**
1. Complete all prerequisite steps before beginning configuration
2. Follow the exact sequence of operations
3. Do not skip validation checkpoints
4. Maintain access to emergency console in case of network loss

**VLAN Implementation Status:** This document covers both currently deployed (Live) and future-planned (Planned) VLANs. Planned VLANs are documented in the reference table for completeness but are not yet active in the environment. Procedures in this SOP apply equally to both; status is noted in the VLAN Reference Table in Section 1.

---

## Table of Contents

1. [Prerequisites and Planning](#1-prerequisites-and-planning)
2. [Physical Infrastructure Preparation](#2-physical-infrastructure-preparation)
3. [Managed Switch Initial Configuration](#3-managed-switch-initial-configuration)
4. [OPNSense Installation and Base Configuration](#4-opnsense-installation-and-base-configuration)
5. [VLAN Interface Creation](#5-vlan-interface-creation)
6. [Switch VLAN Configuration](#6-switch-vlan-configuration)
7. [Interface Assignment and IP Configuration](#7-interface-assignment-and-ip-configuration)
8. [Firewall Rule Configuration](#8-firewall-rule-configuration)
9. [Testing and Validation](#9-testing-and-validation)
10. [Common Failure Modes and Recovery](#10-common-failure-modes-and-recovery)
11. [Post-Implementation Tasks](#11-post-implementation-tasks)
12. [Security Hardening](#12-security-hardening)
- [Appendix A: Troubleshooting Commands](#appendix-a-troubleshooting-commands)
- [Appendix B: VLAN Tag Reference](#appendix-b-vlan-tag-reference)
- [Appendix C: Configuration Checklist](#appendix-c-configuration-checklist)
- [Document Revision History](#document-revision-history)

---

## 1. Prerequisites and Planning

### 1.1 Required Hardware

- OPNSense firewall (dedicated hardware or VM with 2+ NICs)
- Managed 802.1Q VLAN-capable switch (TP-Link TL-SG108E or equivalent)
- Upstream gateway device providing internet connectivity
- Physical console access capability (monitor + keyboard or serial console)
- Workstation for configuration access

### 1.2 Network Architecture and VLAN Reference Table

Before beginning, confirm the full VLAN plan. The table below is the authoritative reference for all VLAN IDs, names, subnets, and gateways used throughout this SOP. When a step says "for each VLAN," refer to this table. Live VLANs are currently deployed; Planned VLANs follow the same procedure when implemented.

| VLAN ID | Name | Subnet | Gateway | Purpose | Status |
|---------|------|--------|---------|---------|--------|
| 1 | LAN | 10.0.0.0/24 | 10.0.0.1 | Infrastructure management | Live |
| 10 | SOC | 10.0.10.0/24 | 10.0.10.1 | Security operations center | Live |
| 20 | Services1 | 10.0.20.0/24 | 10.0.20.1 | Production services (tier 1) | Live |
| 30 | Services2 | 10.0.30.0/24 | 10.0.30.1 | Trusted services (tier 2) | Live |
| 40 | SecurityLab1 | 10.99.0.0/24 | 10.99.0.1 | Isolated attack/lab environment | Live |
| 41 | SecurityLab2 | 10.98.0.0/24 | 10.98.0.1 | Isolated attack/lab environment | Live |
| 50 | DMZ | 10.0.50.0/24 | 10.0.50.1 | Demilitarized zone | Live |
| 51 | SSH Bastion | 10.0.51.0/24 | 10.0.51.1 | SSH bastion host | Planned |
| 60 | Storage | 10.0.60.0/24 | 10.0.60.1 | NAS/storage network | Live |

**Physical Topology:**
```
Internet -> Upstream Gateway -> OPNSense WAN -> OPNSense LAN -> Managed Switch -> Devices
```

**Port Assignment Plan:**
- Ports 1-4: Trunk ports (Proxmox nodes or devices requiring multi-VLAN access)
- Port 5: Trunk port to OPNSense LAN interface
- Ports 6-8: Access ports (untagged VLAN 1 for management workstations)

### 1.3 Critical Configuration Rules

**Rule 1: Subnet Separation**
WAN and LAN interfaces must never be on the same subnet. This creates dual-homed routing conflicts.

**Rule 2: Management Access**
Always maintain at least one method of accessing the OPNSense web interface during configuration changes.

**Rule 3: VLAN Tag Consistency**
VLAN tags must match exactly between switch configuration and OPNSense VLAN interfaces.

**Rule 4: Port Mode Matching**
- Trunk ports (Tagged): For devices that will handle multiple VLANs
- Access ports (Untagged): For devices that operate on a single VLAN only

**Rule 5: PVID Configuration**
Port VLAN ID (PVID) determines which VLAN untagged traffic is assigned to. All ports must have a PVID configured.

**Rule 6: Planned VLANs**
Do not configure switch ports, OPNSense interfaces, or firewall rules for Planned VLANs until hardware is staged and ready. Pre-creating VLAN devices in OPNSense without corresponding switch configuration causes no harm, but incomplete firewall rules on active interfaces can create security gaps.

---

## 2. Physical Infrastructure Preparation

### 2.1 Cable Management

**Step 2.1.1:** Label all ethernet cables before disconnecting existing infrastructure
- Use consistent labeling scheme (device name + port number)
- Document cable paths in topology diagram

**Step 2.1.2:** Prepare console access to OPNSense
- Connect monitor and keyboard to OPNSense hardware, OR
- Configure serial console access
- Verify console access works before proceeding

**Step 2.1.3:** Stage managed switch in accessible location
- Do not connect switch to network yet
- Ensure physical access to all switch ports
- Have switch documentation/manual available

### 2.2 Existing Network State Documentation

**Step 2.2.1:** Document current device IP addresses
```
Device Name | Current IP | MAC Address | Gateway | Purpose
------------|-----------|-------------|---------|--------
[device1]   | x.x.x.x   | xx:xx:xx... | x.x.x.x | [role]
```

**Step 2.2.2:** Test current network connectivity
- Verify internet access from workstation
- Verify access to all managed devices
- Document any existing firewall or routing configuration

---

## 3. Managed Switch Initial Configuration

### 3.1 Factory Default Access

**Step 3.1.1:** Identify switch default IP address
- TP-Link switches typically use 192.168.0.1
- Check label on switch or manufacturer documentation
- Default credentials are usually admin/admin or admin/[blank]

**Step 3.1.2:** Configure workstation for direct switch access
- Disconnect workstation from all networks
- Connect workstation directly to switch (any port except uplink)
- Configure workstation static IP: Address 192.168.0.2, Subnet 255.255.255.0

**Step 3.1.3:** Access switch web interface
- Open browser to http://192.168.0.1
- Login with default credentials
- Navigate to System or Network settings

### 3.2 Switch Management IP Configuration

**Step 3.2.1:** Change switch management IP to match production network
- Navigate to Network -> IP Configuration (or equivalent)
- Configure static IP: 10.0.0.100, Subnet 255.255.255.0, Gateway 10.0.0.1
- Save configuration

**Step 3.2.2:** Verify new IP accessibility
- Change workstation back to DHCP or production IP
- Connect workstation to production network
- Verify connectivity: `ping 10.0.0.100`
- Access web interface at new IP: http://10.0.0.100

> **CHECKPOINT 1:** Can you access the switch at its new management IP? If NO, connect directly to switch and reconfigure management IP.

---

## 4. OPNSense Installation and Base Configuration

### 4.1 OPNSense Installation

**Step 4.1.1:** Install OPNSense on target hardware
- Use latest stable OPNSense release
- Complete basic installation wizard
- Identify physical interface names (igc0, em0, re0, etc.)

**Step 4.1.2:** Initial console configuration
- Access OPNSense console (physical or serial)
- Select Option 1: Assign Interfaces
- Assign WAN interface (upstream network connection)
- Assign LAN interface (switch-facing connection)
- Do not assign VLANs yet

**Step 4.1.3:** Configure WAN interface via console
- Select Option 2: Set interface(s) IP address
- Select WAN interface
- Configure static IP in upstream network: IP [upstream_ip]/[cidr], Gateway [upstream_gateway]
- Enable DHCP on WAN: No

**Step 4.1.4:** Configure temporary LAN interface via console
- Select Option 2: Set interface(s) IP address
- Select LAN interface
- Configure static IP: 10.0.0.1/24
- Do not configure gateway (LAN is internal)
- Enable DHCP on LAN: No (optional: Yes for range 10.0.0.100-10.0.0.200)

**Step 4.1.5:** Verify console configuration
- Display all interface configuration (Option 8)
- Confirm WAN has IP and gateway
- Confirm LAN has IP 10.0.0.1

> **CHECKPOINT 2:** Can you ping the OPNSense LAN IP from your workstation? If NO, verify physical connectivity and workstation IP configuration.

### 4.2 Web Interface Initial Access

**Step 4.2.1:** Access OPNSense web interface
- Connect workstation to LAN side of OPNSense
- Ensure workstation has IP in LAN subnet (10.0.0.x)
- Navigate to https://10.0.0.1
- Accept self-signed certificate warning
- Login with default credentials (root/opnsense)

**Step 4.2.2:** Complete setup wizard
- General: Hostname, Domain, Primary DNS, Secondary DNS
- Time Server: Set timezone, accept default NTP servers
- WAN Interface: Verify matches console settings; uncheck RFC1918/Bogon blocks if upstream is RFC1918
- LAN Interface: Verify matches console settings
- Set root password and document in password manager
- Reload configuration: Yes

**Step 4.2.3:** Verify internet connectivity
- Navigate to System -> Firmware -> Status
- Attempt to check for updates
- If fails: Interfaces -> Diagnostics -> Ping to test upstream gateway
- System -> Gateways: Verify gateway status is "Online"

> **CHECKPOINT 3:** Can OPNSense reach the internet? If NO, verify WAN interface configuration and upstream gateway connectivity.

---

## 5. VLAN Interface Creation

This section uses a standardized procedure. Refer to the VLAN Reference Table in Section 1.2 for all VLAN IDs, names, and subnets. Repeat the procedure for each VLAN listed in the table. Planned VLANs may be pre-created in OPNSense now or deferred until implementation.

### 5.1 Create VLAN Devices in OPNSense

**Step 5.1.1:** Navigate to VLAN configuration
- Interfaces -> Other Types -> VLAN (menu location varies by OPNSense version)

**Step 5.1.2:** For each VLAN in the VLAN Reference Table, create a VLAN device
- Click "+ Add"
- Parent Interface: Select the LAN physical interface (re0, igc1, em1, etc.)
- VLAN Tag: Set to the VLAN ID from the reference table
- VLAN Priority: Best Effort (0, default)
- Description: Set to the VLAN Name from the reference table (e.g., SOC, Services1, DMZ)
- Click Save
- Repeat for each VLAN

> **NOTE:** Device names will follow the pattern `[parent]_vlan[tag]` (e.g., re0_vlan10, re0_vlan20).

**Step 5.1.3:** Verify VLAN creation
- All VLAN devices should appear in the interface list
- Confirm count matches the number of VLANs in the reference table

> **CHECKPOINT 4:** Are all VLAN devices visible in the interface list? If NO, retry VLAN creation ensuring correct parent interface is selected.

---

## 6. Switch VLAN Configuration

### 6.1 Enable 802.1Q VLAN Mode

**Step 6.1.1:** Access switch VLAN configuration
- Login to switch web interface at http://10.0.0.100
- Navigate to VLAN -> 802.1Q VLAN Configuration

**Step 6.1.2:** Enable 802.1Q VLAN
- Select "Enable" radio button
- Click "Apply"
- Wait for switch to apply configuration (10-30 seconds)

**Step 6.1.3:** Verify VLAN mode enabled
- VLAN configuration interface should show VLAN management options
- Default VLAN 1 should be visible

### 6.2 Configure VLAN Membership (Standardized Procedure)

> **CRITICAL:** Follow port configuration exactly as specified. Incorrect trunk/access port configuration is the primary cause of connectivity loss.

For each Live VLAN in the VLAN Reference Table, create and configure VLAN membership using the following standard port policies:

| VLAN Type | Ports 1-4 | Port 5 | Ports 6-8 |
|-----------|-----------|--------|-----------|
| VLAN 1 (LAN/Management) | Untagged | Tagged | Untagged |
| All other VLANs | Tagged | Tagged | Not Member |

**Step 6.2.1:** For each VLAN in the reference table, create VLAN membership
- Enter the VLAN ID from the reference table
- Set VLAN Name to match the reference table name
- Apply the port policy from the table above
- Click "Add/Modify"
- Repeat for each VLAN

**Step 6.2.2:** Verify all VLANs created
- Navigate to VLAN summary page
- Confirm all Live VLANs from the reference table are listed
- Verify port memberships match the policies above

### 6.3 Configure PVID

**Step 6.3.1:** Navigate to PVID settings
- Look for "PVID Configuration" or "Port VLAN ID" menu
- May be under VLAN settings or Port Configuration

**Step 6.3.2:** Set PVID for all ports
- All ports should be set to PVID 1 (Management VLAN)
- This is typically the default configuration
- Verify all ports show PVID = 1
- Click "Apply"

> **CHECKPOINT 5:** Can you still access the switch web interface at its management IP? If NO, port 5 configuration is incorrect - see Recovery Procedure A.

---

## 7. Interface Assignment and IP Configuration

> **CRITICAL:** Complete VLAN 1 (LAN) interface configuration first. This interface maintains management access. Do not proceed to other VLANs until VLAN 1 is confirmed working.

### 7.1 Assign VLAN 1 (LAN) Interface First

**Step 7.1.1:** Assign VLAN 1 interface
- Navigate to Interfaces -> Assignments
- Select re0_vlan1 (or equivalent) from the "Network port" dropdown
- Click "+" to add; new interface will be created (OPT1)
- Click "Save"

**Step 7.1.2:** Configure VLAN 1 (LAN) interface
- Click on the new interface (OPT1)
- Enable: Check "Enable interface"
- Description: LAN or MGMT
- IPv4 Configuration Type: Static IPv4
- IPv4 Address: 10.0.0.1 / 24
- Gateway: None (internal interface)
- Click Save, then Apply Changes

**Step 7.1.3:** Test management access via new VLAN 1 interface
- Open a new browser tab and navigate to https://10.0.0.1
- Verify OPNSense web interface loads
- Keep both tabs open during remaining configuration

> **CHECKPOINT 6:** Can you access OPNSense via https://10.0.0.1? If NO, verify switch port 5 is tagged for VLAN 1 - see Recovery Procedure B.

### 7.2 Assign Remaining VLAN Interfaces (Standardized Procedure)

For each remaining VLAN in the reference table, follow the standard assignment procedure below. Use the Gateway IP from the VLAN Reference Table as the OPNSense interface IP for each VLAN.

**Step 7.2.1:** Assign each remaining VLAN interface
- Return to Interfaces -> Assignments
- Select the VLAN device from the dropdown (e.g., re0_vlan10 for VLAN 10)
- Click "+" - a new OPT interface is created
- Click "Save" after each addition
- Repeat for all remaining VLANs

**Step 7.2.2:** Configure each assigned interface
- Navigate to Interfaces -> [OPT interface]
- Enable: Check "Enable interface"
- Description: Set to the VLAN Name from the reference table
- IPv4 Configuration Type: Static IPv4
- IPv4 Address: Set to the Gateway IP from the reference table
- Subnet: 24
- Gateway: None (all are internal interfaces)
- Click Save, then Apply Changes
- Repeat for each VLAN

> **NOTE:** The Gateway column in the VLAN Reference Table represents the OPNSense interface IP for each VLAN, which also serves as the default gateway for devices on that VLAN.

**Step 7.2.3:** Verify all interfaces
- Navigate to Interfaces -> Overview
- All VLAN interfaces should show "Enabled"
- All interfaces should show the correct IP addresses matching the reference table
- Interface descriptions should match VLAN names

---

## 8. Firewall Rule Configuration

### 8.1 Default Firewall Rule Behavior

- WAN: Block all inbound, allow all outbound
- LAN: Allow all
- New VLAN interfaces: Block all by default

> **NOTE:** Start with permissive rules for initial testing, then restrict based on security requirements.

### 8.2 Firewall Policy Reference

The following table defines the intended firewall policy for each VLAN. Use this as the specification when creating rules in the steps below.

| VLAN | Internet Access | Inter-VLAN | Notes |
|------|----------------|------------|-------|
| 1 - LAN | Yes | All networks | Full management access |
| 10 - SOC | Yes | Read-only to all | SIEM monitoring; Wazuh agent traffic in |
| 20 - Services1 | Yes | Block to LAN (optional) | Production services tier 1 |
| 30 - Services2 | Yes | Block to LAN (optional) | Trusted services tier 2 |
| 40 - SecurityLab1 | Yes | Block all RFC1918 | Isolated; internet egress only |
| 41 - SecurityLab2 | Yes | Block all RFC1918 | Isolated; internet egress only |
| 50 - DMZ | Inbound (ports only) | Block to LAN | Public-facing services; no VMs currently |
| 51 - SSH Bastion | No | LAN SSH pinhole only | Planned - SSH bastion to VLAN 1 only |
| 60 - Storage | No | LAN/MGMT only | NAS; no internet exposure |

### 8.3 Management VLAN (VLAN 1 - LAN) Rules

**Step 8.3.1:** Add rule: Allow Management to All
- Navigate to Firewall -> Rules -> MGMT (or LAN)
- Click "Add"
- Action: Pass, Interface: MGMT, Direction: in, Protocol: any
- Source: MGMT net, Destination: any
- Description: Allow Management VLAN to All Networks
- Click Save, then Apply Changes

### 8.4 SOC VLAN (VLAN 10) Rules

**Step 8.4.1:** Add rule: Allow SOC to All
- Navigate to Firewall -> Rules -> SOC
- Click "Add"
- Action: Pass, Protocol: any
- Source: SOC net, Destination: any
- Description: Allow SOC VLAN monitoring and internet access
- Click Save, then Apply Changes

> **NOTE:** The SOC VLAN requires access to all VLANs for Wazuh agent communication and SIEM log ingestion. Restrict further once baseline monitoring is established.

### 8.5 Services1 VLAN (VLAN 20) Rules

**Step 8.5.1:** Add rule: Allow Services1 to Internet
- Navigate to Firewall -> Rules -> Services1
- Click "Add"
- Action: Pass, Protocol: any
- Source: Services1 net, Destination: any
- Description: Allow Services1 VLAN Internet Access
- Click Save, then Apply Changes

**Step 8.5.2:** Optional: Block Services1 to Management
- Click "Add" (place ABOVE the internet allow rule)
- Action: Block, Protocol: any
- Source: Services1 net, Destination: MGMT net
- Description: Block Services1 Access to Management
- Click Save, then Apply Changes

### 8.6 Services2 VLAN (VLAN 30) Rules

**Step 8.6.1:** Add rule: Allow Services2 to Internet
- Navigate to Firewall -> Rules -> Services2
- Click "Add"
- Action: Pass, Protocol: any
- Source: Services2 net, Destination: any
- Description: Allow Services2 VLAN Internet Access
- Click Save, then Apply Changes

**Step 8.6.2:** Optional: Block Services2 to Management
- Click "Add" (place ABOVE the internet allow rule)
- Action: Block, Protocol: any
- Source: Services2 net, Destination: MGMT net
- Description: Block Services2 Access to Management
- Click Save, then Apply Changes

### 8.7 Security Lab VLANs (VLAN 40 and VLAN 41) Rules

> **CRITICAL:** Security lab VLANs must be isolated from all production networks. Apply identical rules to both SecurityLab1 and SecurityLab2.

**Step 8.7.1:** Add rule: Block Security Lab to RFC1918 (apply to both VLANs)
- Navigate to Firewall -> Rules -> SecurityLab1 (then repeat for SecurityLab2)
- Click "Add" (this rule must be ABOVE the internet allow rule)
- Action: Block, Protocol: any
- Source: SecurityLab1 net
- Destination: RFC1918 alias (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- Description: Block Security Lab to Internal Networks
- Click Save

**Step 8.7.2:** Add rule: Allow Security Lab to Internet Only (apply to both VLANs)
- Click "Add" (BELOW the RFC1918 block rule)
- Action: Pass, Protocol: any
- Source: SecurityLab1 net, Destination: any
- Description: Allow Security Lab Internet Access Only
- Click Save, then Apply Changes
- Repeat both rules for SecurityLab2 (VLAN 41)

### 8.8 DMZ VLAN (VLAN 50) Rules

> **NOTE:** The DMZ VLAN is live but currently has no VMs or containers assigned. Configure firewall rules now to establish the correct policy before any services are deployed.

**Step 8.8.1:** Add rule: Block DMZ to LAN
- Navigate to Firewall -> Rules -> DMZ
- Click "Add" (place ABOVE any allow rules)
- Action: Block, Protocol: any
- Source: DMZ net, Destination: LAN net (or RFC1918 alias)
- Description: Block DMZ to Internal Networks
- Click Save

**Step 8.8.2:** Add rule: Allow DMZ internet inbound (as needed per service)
- Inbound access from the internet to DMZ services is controlled via Firewall -> NAT -> Port Forward
- Create port forward rules per service when VMs are deployed
- Click Save, then Apply Changes

### 8.9 Storage VLAN (VLAN 60) Rules

**Step 8.9.1:** Add rule: Allow Storage to Management only
- Navigate to Firewall -> Rules -> Storage
- Click "Add"
- Action: Pass, Protocol: any
- Source: Storage net, Destination: MGMT net
- Description: Allow Storage Access to Management
- Click Save

**Step 8.9.2:** Add rule: Block Storage to Internet and all other networks
- Click "Add" (place BELOW the management allow rule)
- Action: Block, Protocol: any
- Source: Storage net, Destination: any
- Description: Block Storage VLAN All Other Access
- Click Save, then Apply Changes

### 8.10 Planned VLANs (VLAN 51 - SSH Bastion)

> **NOTE:** SSH Bastion (VLAN 51) is a Planned VLAN. Do not create firewall rules until the VLAN is implemented and tested. Policy specification is documented in the Firewall Policy Reference table (Section 8.2).

### 8.11 Outbound NAT Configuration

**Step 8.11.1:** Verify Outbound NAT mode
- Navigate to Firewall -> NAT -> Outbound
- Mode should be "Automatic outbound NAT rule generation"
- If mode is Manual, switch to Automatic
- Click Save, then Apply Changes

**Step 8.11.2:** Verify NAT rules generated
- All Live VLAN subnets should have automatic NAT rules
- Rules should translate internal IPs to WAN IP
- If rules are missing: switch to Manual mode, then back to Automatic, save and apply

---

## 9. Testing and Validation

### 9.1 Management VLAN Connectivity

**Test 9.1.1:** Workstation to OPNSense
- From workstation on management VLAN (10.0.0.x): `ping 10.0.0.1`
- Expected: Success
- If fails: Check switch port configuration and workstation IP

**Test 9.1.2:** Workstation to Internet
- From workstation on management VLAN: `ping 8.8.8.8`
- Expected: Success
- If fails: Check OPNSense WAN connectivity and firewall rules

**Test 9.1.3:** DNS Resolution
- From workstation on management VLAN: `nslookup google.com`
- Expected: Success with valid IP
- If fails: Check DNS configuration in OPNSense

### 9.2 Inter-VLAN Routing

**Test 9.2.1:** Management to each VLAN
- From device on VLAN 1 (10.0.0.x), ping the gateway of each VLAN
- Expected: Success (management has full access per firewall policy)
- If unexpected result: Check firewall rules on MGMT interface

**Test 9.2.2:** Each VLAN to Management
- From device on each VLAN, ping 10.0.0.1
- Expected result varies by VLAN policy (see Firewall Policy Reference in Section 8.2)
- If unexpected result: Check firewall rules for that VLAN interface

### 9.3 Security Lab Isolation

**Test 9.3.1:** Security Lab to Management VLAN
- From device on VLAN 40 or 41: `ping 10.0.0.1`
- Expected: Block / Timeout
- **If succeeds: CRITICAL - firewall rules are incorrect, fix immediately**

**Test 9.3.2:** Security Lab to Internet
- From device on VLAN 40 or 41: `ping 8.8.8.8`
- Expected: Success
- If fails: Check firewall rules and NAT configuration

### 9.4 VLAN Tag Verification

**Test 9.4.1:** Packet capture on OPNSense
- Navigate to Interfaces -> Diagnostics -> Packet Capture
- Interface: LAN physical interface (re0, etc.)
- Start capture, generate traffic from different VLANs, stop capture
- Expected: 802.1Q VLAN tags visible in packet headers
- If no VLAN tags: Switch is not properly tagging traffic

**Test 9.4.2:** Switch port statistics
- Navigate to switch Monitoring -> Port Statistics
- Verify packet counts incrementing on all active ports
- Expected: Low or zero error counts
- If high errors: Check cable quality or port configuration

---

## 10. Common Failure Modes and Recovery

### Recovery Procedure A: Lost Access to Switch After VLAN Configuration

**Symptom:** Cannot access switch web interface after enabling 802.1Q VLANs or configuring port 5.

**Root Cause:** Port 5 (OPNSense uplink) is configured as Tagged for VLAN 1, but OPNSense is not yet sending tagged VLAN 1 traffic.

**Recovery Steps:**
1. Connect workstation directly to switch on a port other than port 5
2. Temporarily change that port to Untagged VLAN 1
3. Modify port 5 configuration: change from Tagged to Untagged for VLAN 1
4. Save and apply; test OPNSense connectivity
5. Once OPNSense VLAN 1 interface is configured and working, change port 5 back to Tagged

**Prevention:** Do not configure port 5 as Tagged for VLAN 1 until after OPNSense VLAN 1 interface is fully configured and tested.

### Recovery Procedure B: Lost Access to OPNSense After VLAN 1 Configuration

**Symptom:** Cannot access OPNSense web interface at 10.0.0.1 after creating VLAN 1 interface.

**Root Cause:** Switch port 5 is still Untagged for VLAN 1, but OPNSense is now sending tagged VLAN 1 traffic.

**Recovery Steps:**
1. Access OPNSense console (physical or serial)
2. Select Option 8: Shell
3. Run `ifconfig` to verify VLAN 1 interface exists and has correct IP
4. Access switch web interface and configure port 5 as Tagged for VLAN 1
5. Test access to OPNSense at 10.0.0.1

**Prevention:** Verify switch port 5 configuration matches OPNSense VLAN configuration before applying changes.

### Recovery Procedure C: No Internet Access After Configuration

**Cause 1: Firewall Rules Block Traffic**
- Navigate to Firewall -> Rules -> [VLAN interface]
- Verify "Allow to any" rule exists and rule order is correct (block rules above allow rules)

**Cause 2: Outbound NAT Not Configured**
- Navigate to Firewall -> NAT -> Outbound
- Verify rules exist for all VLAN subnets; switch to Automatic mode if missing

**Cause 3: Default Gateway Not Set on Client**
- Verify client default gateway is set to the VLAN gateway IP from the reference table

**Cause 4: DNS Not Configured**
- Test with `ping 8.8.8.8` (should work if routing is correct)
- If DNS fails, configure DNS servers on client or enable DNS forwarder in OPNSense

**Diagnostic Commands from OPNSense shell:**
```bash
ping -c 4 8.8.8.8       # Test WAN connectivity
host google.com          # Test DNS resolution
pfctl -ss                # View firewall states
netstat -rn              # View routing table
```

### Recovery Procedure D: VLAN Connectivity Between VMs Fails

**Symptom:** VMs on Proxmox nodes cannot communicate on VLANs despite correct OPNSense and switch configuration.

**Root Cause:** Proxmox bridge is not VLAN-aware or VMs are not tagged with the correct VLAN.

**Recovery Steps:**
1. Access Proxmox web interface
2. Navigate to node -> System -> Network
3. Select bridge interface (vmbr0 or similar)
4. Enable "VLAN aware" checkbox and click "Apply Configuration"
5. For each VM: navigate to VM -> Hardware -> Network Device, set VLAN Tag to the appropriate ID
6. Restart VM networking or reboot VM

**Prevention:** Configure Proxmox bridges as VLAN-aware before creating VLAN-tagged VMs.

### Recovery Procedure E: Complete Network Loss

**Symptom:** Cannot access any network devices including OPNSense and switch.

**Recovery Steps:**
1. Connect to OPNSense via console (physical or serial)
2. Run `ifconfig` to verify interface status
3. Check for IP conflicts (duplicate IPs on WAN/LAN)
4. If necessary: Option 2 on console to reconfigure LAN with known-good IP
5. Factory reset switch if needed: hold physical reset button 10+ seconds
6. Restart from Section 3 with confirmed-working IPs

**Prevention:**
- Maintain console access at all times
- Document all IP changes before applying
- Test configuration changes incrementally
- Keep backup configuration files

---

## 11. Post-Implementation Tasks

### 11.1 Configuration Backup

**Step 11.1.1:** Backup OPNSense configuration
- Navigate to System -> Configuration -> Backups
- Click "Download configuration"
- Save file with date stamp: `opnsense-config-YYYY-MM-DD.xml`
- Store in secure location off-device

**Step 11.1.2:** Backup switch configuration
- Access switch web interface
- Navigate to System or Management
- Look for "Backup Configuration" or "Export Configuration"
- Save with date stamp and store securely

### 11.2 Documentation Updates

**Step 11.2.1:** Update network diagram
- Document all VLAN assignments and IP address allocations
- Document firewall rules per VLAN
- Document port assignments on switch

**Step 11.2.2:** Create device inventory
```
Device | VLAN | IP Address | MAC Address | Port | Purpose
-------|------|-----------|-------------|------|--------
[name] | [#]  | [IP]      | [MAC]       | [#]  | [role]
```

**Step 11.2.3:** Document access credentials
- OPNSense root password
- Switch admin password
- Store in secure password manager

### 11.3 Monitoring and Alerts

**Step 11.3.1:** Configure OPNSense logging
- Navigate to System -> Settings -> Logging
- Enable remote logging if using centralized log server (e.g., Wazuh/ELK on VLAN 10)
- Set log retention period
- Enable logging for firewall rules

**Step 11.3.2:** Configure switch logging (if available)
- Enable SNMP for monitoring
- Configure syslog destination if available

**Step 11.3.3:** Test alert mechanisms
- Generate test firewall block
- Verify logs capture event
- Test any configured alerts

---

## 12. Security Hardening

### 12.1 OPNSense Security

**Step 12.1.1:** Disable unnecessary services
- Navigate to System -> Settings -> Administration
- Disable SSH if not required for remote management
- Enable "Disable web GUI redirect rule"
- Configure HTTPS-only access

**Step 12.1.2:** Configure firewall aliases for rule management
- Create aliases for commonly used IPs/networks (e.g., RFC1918)
- Use aliases in firewall rules for easier management
- Navigate to Firewall -> Aliases

**Step 12.1.3:** Enable intrusion detection
- Navigate to Services -> Intrusion Detection
- Enable IDS/IPS if system resources permit
- Configure rule sets appropriate to your environment

### 12.2 Switch Security

**Step 12.2.1:** Change default credentials
- Change admin password from default
- Use strong password (20+ characters)
- Document in secure password manager

**Step 12.2.2:** Disable unused ports
- Navigate to Port Configuration
- Disable all unused switch ports
- This prevents unauthorized physical access

**Step 12.2.3:** Configure port security (if available)
- Enable MAC address filtering per port
- Limit number of MAC addresses per port
- Configure violation actions

---

## Appendix A: Troubleshooting Commands

### OPNSense Console Commands

```bash
# View all interface configurations
ifconfig

# View routing table
netstat -rn

# Test connectivity
ping -c 4 [destination]

# DNS lookup
host [hostname]
nslookup [hostname]

# View firewall states
pfctl -ss

# View firewall rules
pfctl -sr

# View NAT rules
pfctl -sn

# Restart networking
/etc/rc.reload_interfaces

# View logs
clog -f /var/log/system.log
clog -f /var/log/filter.log

# Packet capture
tcpdump -i [interface] -n

# VLAN-specific capture
tcpdump -i [interface] -n vlan [vlan_id]
```

### Switch Diagnostic Commands (if CLI available)

```bash
show vlan
show interfaces status
show mac address-table
show interfaces counters
```

---

## Appendix B: VLAN Tag Reference

The following table is the authoritative reference for all VLANs in this environment. This mirrors Section 1.2 and is reproduced here for quick reference during troubleshooting.

| VLAN ID | Name | Subnet | Gateway | Purpose | Status |
|---------|------|--------|---------|---------|--------|
| 1 | LAN | 10.0.0.0/24 | 10.0.0.1 | Infrastructure management | Live |
| 10 | SOC | 10.0.10.0/24 | 10.0.10.1 | Security operations center | Live |
| 20 | Services1 | 10.0.20.0/24 | 10.0.20.1 | Production services (tier 1) | Live |
| 30 | Services2 | 10.0.30.0/24 | 10.0.30.1 | Trusted services (tier 2) | Live |
| 40 | SecurityLab1 | 10.99.0.0/24 | 10.99.0.1 | Isolated attack/lab environment | Live |
| 41 | SecurityLab2 | 10.98.0.0/24 | 10.98.0.1 | Isolated attack/lab environment | Live |
| 50 | DMZ | 10.0.50.0/24 | 10.0.50.1 | Demilitarized zone | Live |
| 51 | SSH Bastion | 10.0.51.0/24 | 10.0.51.1 | SSH bastion host | Planned |
| 60 | Storage | 10.0.60.0/24 | 10.0.60.1 | NAS/storage network | Live |

---

## Appendix C: Configuration Checklist

### Pre-Implementation Checklist

- [ ] Hardware inventory completed
- [ ] VLAN Reference Table reviewed and confirmed
- [ ] Port assignment plan documented
- [ ] Console access to OPNSense verified
- [ ] Switch default credentials confirmed
- [ ] Existing network state documented
- [ ] Backup configuration files saved
- [ ] Emergency recovery plan reviewed

### Switch Configuration Checklist

- [ ] Switch accessible at default IP
- [ ] Management IP configured and accessible at 10.0.0.100
- [ ] 802.1Q VLAN mode enabled
- [ ] VLAN 1 (LAN) configured with correct port memberships
- [ ] VLAN 10 (SOC) configured with correct port memberships
- [ ] VLAN 20 (Services1) configured with correct port memberships
- [ ] VLAN 30 (Services2) configured with correct port memberships
- [ ] VLAN 40 (SecurityLab1) configured with correct port memberships
- [ ] VLAN 41 (SecurityLab2) configured with correct port memberships
- [ ] VLAN 50 (DMZ) configured with correct port memberships
- [ ] VLAN 60 (Storage) configured with correct port memberships
- [ ] PVID configured for all ports
- [ ] Switch configuration backed up

### OPNSense Configuration Checklist

- [ ] WAN interface configured with internet access
- [ ] LAN interface configured for initial management access
- [ ] Web interface accessible
- [ ] VLAN 1 device created and interface configured (10.0.0.1)
- [ ] VLAN 10 device created and interface configured (10.0.10.1)
- [ ] VLAN 20 device created and interface configured (10.0.20.1)
- [ ] VLAN 30 device created and interface configured (10.0.30.1)
- [ ] VLAN 40 device created and interface configured (10.99.0.1)
- [ ] VLAN 41 device created and interface configured (10.98.0.1)
- [ ] VLAN 50 device created and interface configured (10.0.50.1)
- [ ] VLAN 60 device created and interface configured (10.0.60.1)
- [ ] Firewall rules configured for LAN (VLAN 1)
- [ ] Firewall rules configured for SOC (VLAN 10)
- [ ] Firewall rules configured for Services1 (VLAN 20)
- [ ] Firewall rules configured for Services2 (VLAN 30)
- [ ] Firewall rules configured for SecurityLab1 (VLAN 40)
- [ ] Firewall rules configured for SecurityLab2 (VLAN 41)
- [ ] Firewall rules configured for DMZ (VLAN 50)
- [ ] Firewall rules configured for Storage (VLAN 60)
- [ ] Outbound NAT verified for all Live VLANs
- [ ] OPNSense configuration backed up

### Testing Checklist

- [ ] VLAN 1 (LAN) connectivity verified
- [ ] VLAN 10 (SOC) connectivity verified
- [ ] VLAN 20 (Services1) connectivity verified
- [ ] VLAN 30 (Services2) connectivity verified
- [ ] VLAN 40 (SecurityLab1) connectivity verified
- [ ] VLAN 41 (SecurityLab2) connectivity verified
- [ ] VLAN 50 (DMZ) connectivity verified
- [ ] VLAN 60 (Storage) connectivity verified
- [ ] SecurityLab1 and SecurityLab2 isolation from production verified
- [ ] Inter-VLAN routing tested per firewall policy table (Section 8.2)
- [ ] DNS resolution tested on all VLANs
- [ ] VLAN tags visible in packet captures
- [ ] Switch port statistics normal (low error counts)
- [ ] All devices accessible on correct VLANs

### Planned VLANs (Not Yet Implemented)

- [ ] VLAN 51 (SSH Bastion) - not yet implemented

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-11 | Network Infrastructure | Initial comprehensive SOP based on complete implementation history and failure mode analysis |
| 2.0 | 2026-04-21 | Network Infrastructure | Updated all IP addressing to current scheme (10.0.x.x); renamed VLAN 20 to Services1 and VLAN 30 to Services2; added VLAN 10 (SOC - Live), VLAN 50 (DMZ - Live, no current VMs), VLAN 51 (SSH Bastion - Planned); restructured per-VLAN procedures to standardized generic procedure with VLAN Reference Table; added Firewall Policy Reference table; converted to Markdown for GitHub |

---

*ryanshomelab.org | Internal Documentation | Not for Distribution*

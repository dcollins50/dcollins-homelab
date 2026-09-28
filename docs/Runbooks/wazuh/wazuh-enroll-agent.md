---
name: wazuh-enroll-agent
type: runbook
tool: wazuh
---

Install and enroll a new Wazuh agent against [[wazuh-manager]] (10.0.10.11). Written up from the actual rollout that enrolled 8 agents in April 2026 (VM 401, all 4 Proxmox hosts, services-host, soc-stack, Jetson), not from vendor docs, so the steps below are what actually worked here, including a correction discovered mid-rollout.

## Prerequisites

- OPNSense firewall rule permitting the agent's subnet to reach wazuh-manager on TCP/UDP 1514-1515. This homelab uses reusable aliases (`Wazuh_Manager`, and per-VLAN agent aliases) rather than host-specific rules, add the new agent's VLAN to the relevant alias if it isn't already covered.
- TLS enrollment on the manager side (`wazuh-authd`, port 1515) already configured, this was a one-time manager setup, not repeated per agent.

## Procedure

**1. Install the agent package.** On Debian-based hosts (all Proxmox nodes, most VMs so far):
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --dearmor | sudo tee /usr/share/keyrings/wazuh.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update && sudo apt install wazuh-agent -y
```
On the Jetson (arm64), the same commands resolved the correct architecture automatically, no separate repo needed.

**2. Configure the client block.** Edit `/var/ossec/etc/ossec.conf`:
```xml
<client>
  <server>
    <address>10.0.10.11</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
  <enrollment>
    <enabled>yes</enabled>
    <manager_address>10.0.10.11</manager_address>
    <port>1515</port>
  </enrollment>
</client>
```
**Do not add `server_address` or `ssl_manager_ca` tags inside `<enrollment>`.** The first agent enrolled (VM 401) was configured with an extra `ssl_manager_ca` line pointing at a manually-copied manager cert; it worked, but turned out unnecessary; Wazuh 4.14.4's enrollment negotiates this itself. Every agent after VM 401 used the simpler block above with no manual cert copy step at all.

**3. Test the config before starting the service:**
```bash
sudo /var/ossec/bin/wazuh-agentd -t 2>&1
```
No output means it's clean.

**4. Enable and start:**
```bash
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

**5. Confirm enrollment on the agent:**
```bash
sudo grep "Valid key received" /var/ossec/logs/ossec.log
```

**6. Confirm on the manager**, before moving to the next host:
```bash
sudo /var/ossec/bin/agent_control -l
```
The new agent should show `Active`.

## Notes

- **Hostname**: the agent registers under the machine's hostname at enrollment time. VM 401 enrolled as `admin-Standard-PC-Q35-ICH9-2009` (the default QEMU machine name) because it was never renamed first. This is still unfixed as of the Sep 24, 2026 session (open item, see [[session-2026-09-24-alert-engineering-baseline]]) and now has 13 alerts of its own from a CIS sshd rule. Rename the VM hostname to something meaningful *before* enrolling to avoid this.
- **Skipped on purpose**: kalshi-mm and VM 700 were both deliberately not enrolled during the April rollout. kalshi-mm no longer exists (destroyed, per [[homelab-soc]]); VM 700 doesn't exist either.
- **Do not enroll**: metasploitable2, DVWA, malware-win11 — intentionally compromised/isolated lab targets, no agent by design.

## Related
- [[wazuh-manager]]
- [[wazuh-check-agent-status]]
- [[wazuh-restart-agent-confirm-scan]]

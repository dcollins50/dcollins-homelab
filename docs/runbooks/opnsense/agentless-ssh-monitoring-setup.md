# Runbook: Agentless SSH Monitoring Setup (OPNSense)

| Field | Value |
| --- | --- |
| Applies to | OPNSense, Wazuh Manager |
| Category | SOC Integration |
| Author | Daniel Collins |

---

## When to use this

OPNSense can't run a native Wazuh agent (it's not a general-purpose Linux host Wazuh supports directly), so it's monitored via Wazuh's agentless SSH capability instead: the Wazuh Manager periodically SSHes in and runs integrity/config checks rather than a resident agent reporting continuously. Use this runbook for initial setup, or when the check stops working and needs re-established.

---

## How it works

Wazuh Manager's agentless module runs on a schedule, using a locally-stored SSH key to connect to OPNSense's management interface, run a defined check (e.g. the built-in `ssh_integrity_check_bsd` command set), and diff the output against the previous run to detect changes. This depends on:

- Passwordless SSH access from Wazuh Manager to OPNSense for the specific check account
- That account being able to run the check commands without a password prompt (sudo/doas NOPASS scoped to what's needed)
- The Wazuh Manager's `ossec.conf` agentless block being correctly configured with the right host, port, and check type

---

## Setup steps

### On OPNSense

1. Ensure SSH access is enabled and reachable from the Wazuh Manager's VLAN (this requires an explicit firewall rule, see [Add a Firewall Rule](add-firewall-rule.md), agentless SSH doesn't get an exception to default-deny).
2. Generate or confirm an SSH keypair for the Wazuh Manager to use, and install the public key for the account the agentless check will authenticate as.
3. **Add the account to the passlist** so the check can run its command(s) without an interactive password/passphrase prompt, e.g. an entry equivalent to:
   ```
   <check-account>@<opnsense-mgmt-ip> NOPASS
   ```
   This is the step most likely to be missed or to get silently reset, see Troubleshooting below.

### On Wazuh Manager

1. Edit the agentless block in `ossec.conf` (or the equivalent agentless configuration file):
   - Host: OPNSense's management IP
   - Port: the SSH port in use
   - Type: the check type being used (e.g. `ssh_integrity_check_bsd` for a BSD-based system like OPNSense)
   - Frequency: how often the check runs
2. Restart the Wazuh Manager service (or reload the agentless config, depending on version) to pick up the change.
3. Confirm the check runs and reports success in the Wazuh Manager logs, rather than just assuming the config is correct because it saved without error.

---

## Troubleshooting

**Symptom: agentless check stops passing after previously working.**

This has happened in this environment before, root cause was the `.passlist` entry going missing after a manager restart (the NOPASS entry didn't persist). If the check starts failing:

1. Confirm the `.passlist` entry (or equivalent) is still present on the account used for the check, don't assume it's still there just because it was configured once, this specific entry has been lost before.
2. Re-add it if missing: `<check-account>@<opnsense-mgmt-ip> NOPASS`
3. Re-run the check manually if possible, or wait for the next scheduled run, and confirm in the Wazuh Manager logs that it now reports success (e.g. `Test passed for '<check-type>'`) rather than just assuming the fix worked.

**Symptom: check has never worked / initial setup.**

1. Confirm SSH connectivity itself works independent of Wazuh: can you manually SSH from the Wazuh Manager host to OPNSense as the check account?
2. Confirm the firewall actually permits this SSH path, this is a common point of failure alongside the passlist issue.
3. Confirm the account can run the check command without a password prompt when tested manually, before assuming the problem is in Wazuh's config.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Add a Firewall Rule](add-firewall-rule.md)

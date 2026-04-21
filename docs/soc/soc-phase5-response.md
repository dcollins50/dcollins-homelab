**Phase 5: Active Response and Forensic Collection**   |   April 19, 2026

**HOMELAB SOC**

Phase 5: Active Response and Forensic Collection

*Learning Exercise and Standard Operating Procedure*

| **Document Date** | April 19, 2026 |
| --- | --- |
| **Phase** | 5 of 5 |
| **Author** | Daniel Collins |
| **Prerequisite** | Phases 1 through 4 complete |
| **Ansible Control Node** | VM 401 (ubuntu, pve-services) |
| **Forensic Storage** | soc-stack /opt/forensics (VM 600) |
| **Estimated Time** | 6 to 10 hours |

# **1. Purpose and Learning Objectives**

> **Status:** Planned
>
> The Ansible active response framework and Proxmox API isolation have not yet been deployed. This document defines the target architecture and implementation procedure.

---



Phases 1 through 4 cover detection and investigation. Phase 5 covers response. Active response is automated action taken by the SIEM when specific alert conditions are met. Forensic collection is the practice of capturing the state of a compromised system before that state changes. Together they form the response half of a complete SOC capability.

| **WHY** | Speed matters in incident response. The longer an attacker remains undetected, the more damage they can do and the harder forensic reconstruction becomes. Automated response compresses the time between detection and containment from hours or days to seconds. Automated forensic collection ensures that evidence is preserved even if you do not see the alert until the next morning. |
| --- | --- |

## **1.1 About Ansible in This Context**

Ansible is an open source automation platform that executes tasks on remote hosts over SSH. It requires no agent on the target host. A playbook is a YAML file that describes a sequence of tasks. In this phase, Ansible serves two purposes: it runs forensic collection playbooks on affected hosts when Wazuh triggers an active response, and it provides a reusable, documented, and version-controlled automation layer that is more maintainable than ad hoc shell scripts.

Ansible is a standard tool in both DevOps and security operations. Building familiarity with it in this context is directly transferable to enterprise environments.

## **1.2 Learning Objectives**

- Install and configure Ansible on VM 401 as a dedicated control node

- Distribute an SSH keypair to all managed hosts for agentless automation

- Write Ansible playbooks for forensic data collection

- Configure Wazuh active response to trigger Ansible playbooks on alert

- Implement four active response actions: IP block, user disable, service restart, and host isolation

- Establish a forensic storage directory on soc-stack with appropriate permissions

## **1.3 Success Criteria**

This phase is complete when Ansible is operational on VM 401 with SSH access to all managed hosts, a forensic collection playbook runs successfully against a test host and deposits output on soc-stack, and at least two Wazuh active response rules are triggering correctly in test conditions.

# **2. Architecture**

## **2.1 Component Overview**

| **Component** | **Host** | **Role** |
| --- | --- | --- |
| Ansible control node | VM 401 | Executes playbooks against managed hosts. SSH keypair stored here. |
| Wazuh active response | VM 601 | Detects alert conditions and triggers response scripts. Scripts call Ansible on VM 401. |
| Forensic storage | VM 600 | /opt/forensics directory. Ansible playbooks SCP collected data here. |
| Managed hosts | All agents | Targets for forensic collection. Ansible connects over SSH using the dedicated keypair. |
| OPNSense | 10.0.0.1 | Target for IP block active response via OPNSense API. |
| Proxmox API | All nodes | Target for host isolation active response. Proxmox API disconnects VM network interface. |

## **2.2 Response Flow**

When an alert fires that has an active response rule configured, the sequence is:

- Wazuh analysisd evaluates the alert against active response rules

- If a match is found, wazuh-execd runs the configured response script on the manager

- The response script SSHes to VM 401 and triggers the appropriate Ansible playbook

- Ansible executes the playbook against the affected host or network device

- Forensic data is collected and deposited to /opt/forensics on soc-stack

- A case note is added to the relevant DFIR-IRIS case via the API

# **3. Prerequisites**

- VM 401 (ubuntu, pve-services) running Ubuntu 24.04 with SSH access

- VM 401 enrolled as a Wazuh agent

- SSH access from VM 601 (wazuh-manager) to VM 401 for triggering playbooks

- All managed hosts accessible via SSH from VM 401

- OPNSense API access enabled (System > Settings > Administration > API)

- Proxmox API token with VM.Config.Network permission on all nodes

- /opt/forensics directory created on soc-stack with appropriate ownership

# **4. Procedure: Ansible Setup**

## **4.1 Install Ansible on VM 401**

sudo apt update && sudo apt install ansible python3-pip -y

ansible --version

Verify Ansible is at version 2.15 or above.

## **4.2 Generate the Ansible SSH Keypair**

Generate a dedicated ed25519 keypair on VM 401. This key is used exclusively for Ansible automation. Keeping it separate from your personal SSH key means you can rotate or revoke it without affecting your own access.

ssh-keygen -t ed25519 -f ~/.ssh/ansible_id -N '' -C 'ansible-control-vm401'

cat ~/.ssh/ansible_id.pub

Copy the public key output. You will add it to each managed host in the next step.

## **4.3 Distribute the Public Key to All Managed Hosts**

On each managed host (VMs 200, 290, 401 self, 600, 601, and all four Proxmox nodes), add the Ansible public key to the authorized_keys file. Use a restricted command to limit what the key can do if it is ever compromised:

On each target host:

echo 'no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 <key> ansible-control-vm401' >> ~/.ssh/authorized_keys

Alternatively, use ssh-copy-id from VM 401 to distribute the key interactively:

ssh-copy-id -i ~/.ssh/ansible_id.pub ryan@<target_host_ip>

| **NOTE** | The no-port-forwarding,no-X11-forwarding,no-agent-forwarding prefix restricts what the key can be used for if an attacker obtains it. It cannot be used to tunnel traffic or forward agents. For forensic collection this is appropriate since the playbooks only need to run commands and copy files. |
| --- | --- |

## **4.4 Create the Ansible Inventory**

sudo mkdir -p /etc/ansible

sudo nano /etc/ansible/hosts

Inventory contents:

[soc]

soc-stack        ansible_host=10.0.10.10

wazuh-manager    ansible_host=10.0.10.11

[vms]

services-host    ansible_host=10.0.20.30

kalshi-mm        ansible_host=10.0.20.50

ubuntu           ansible_host=10.0.30.40

[proxmox_nodes]

pve-gateway      ansible_host=10.0.0.10

pve-services     ansible_host=10.0.0.11

pve-env1         ansible_host=10.0.0.12

pve-SOC         ansible_host=10.0.0.13

[all:vars]

ansible_user=ryan

ansible_ssh_private_key_file=~/.ssh/ansible_id

ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'

Verify connectivity to all hosts:

ansible all -m ping

All hosts should return pong. Investigate any that do not before proceeding.

## **4.5 Set Up the Forensic Storage Directory**

On soc-stack (VM 600), create the forensic storage directory:

sudo mkdir -p /opt/forensics

sudo chown ryan:ryan /opt/forensics

sudo chmod 750 /opt/forensics

Verify that VM 401 can write to this directory via SSH:

ssh ryan@10.0.10.10 'echo test > /opt/forensics/connectivity_test.txt && cat /opt/forensics/connectivity_test.txt'

# **5. Procedure: Forensic Collection Playbook**

## **5.1 Playbook Overview**

The forensic collection playbook captures a point-in-time snapshot of a host's state. It collects process lists, network connections, open files, logged-in users, recent authentication events, and running container state. All output is timestamped and compressed before being copied to soc-stack.

## **5.2 Create the Playbook**

mkdir -p /opt/ansible/playbooks

nano /opt/ansible/playbooks/forensic_collect.yml

Playbook contents:

---

- name: Forensic Collection

  hosts: "{{ target_host }}"

  gather_facts: yes

  vars:

    case_id: "{{ case_id | default('UNKNOWN') }}"

    ts: "{{ ansible_date_time.iso8601_basic_short }}"

    out_dir: "/tmp/forensics_{{ ts }}"

    store: "ryan@10.0.10.10:/opt/forensics/{{ case_id }}_{{ ts }}"

  tasks:

    - name: Create staging directory

      file: path={{ out_dir }} state=directory mode=0700

    - name: Collect running processes

      shell: ps auxf > {{ out_dir }}/processes.txt

    - name: Collect network connections

      shell: ss -tulpna > {{ out_dir }}/network.txt

    - name: Collect open files

      shell: lsof -n 2>/dev/null > {{ out_dir }}/open_files.txt || true

    - name: Collect logged-in users

      shell: who -a > {{ out_dir }}/users.txt && last -n 50 >> {{ out_dir }}/users.txt

    - name: Collect recent auth log

      shell: journalctl -u ssh --since '2 hours ago' > {{ out_dir }}/auth.txt 2>/dev/null || grep -i 'auth\|sshd' /var/log/syslog | tail -200 > {{ out_dir }}/auth.txt || true

    - name: Collect Docker container state

      shell: docker ps -a > {{ out_dir }}/docker.txt 2>/dev/null || echo 'Docker not running' > {{ out_dir }}/docker.txt

    - name: Collect system info

      shell: uname -a > {{ out_dir }}/sysinfo.txt && uptime >> {{ out_dir }}/sysinfo.txt && df -h >> {{ out_dir }}/sysinfo.txt

    - name: Compress collection

      archive: path={{ out_dir }} dest={{ out_dir }}.tar.gz format=gz

    - name: Copy to forensic store

      fetch:

        src: "{{ out_dir }}.tar.gz"

        dest: "/tmp/forensics_staging/"

        flat: yes

    - name: Cleanup staging

      file: path={{ out_dir }} state=absent

      file: path={{ out_dir }}.tar.gz state=absent

## **5.3 Test the Playbook**

ansible-playbook /opt/ansible/playbooks/forensic_collect.yml \

  -e 'target_host=ubuntu case_id=TEST-001'

Verify that the output archive appears in /opt/forensics on soc-stack after the playbook completes.

# **6. Procedure: Wazuh Active Response**

## **6.1 Active Response Concepts**

Wazuh active response works through two configuration sections in ossec.conf: a command definition that specifies what script to run, and an active-response block that ties the command to a rule condition. When the rule fires, Wazuh executes the command script.

## **6.2 Response Action 1: IP Block via OPNSense**

When Wazuh detects suspicious external traffic such as a port scan or brute force attempt, this response adds a firewall block rule via the OPNSense API.

Create the response script on wazuh-manager:

sudo nano /var/ossec/active-response/bin/opnsense-block.sh

#!/bin/bash

ACTION=$1

USER=$2

IP=$3

OPNSENSE_KEY=$(cat /etc/wazuh-opnsense/api.key)

OPNSENSE_SECRET=$(cat /etc/wazuh-opnsense/api.secret)

OPNSENSE_URL='https://10.0.0.1'

ALIAS='wazuh_blocked'

if [ "$ACTION" = 'add' ]; then

  curl -sk -u "$OPNSENSE_KEY:$OPNSENSE_SECRET" \

    -X POST "$OPNSENSE_URL/api/firewall/alias_util/add/$ALIAS" \

    -H 'Content-Type: application/json' \

    -d "{\"address\":\"$IP\"}"

fi

In OPNSense, create a host alias named wazuh_blocked and add it to a block rule on the WAN interface. The script populates this alias with offending IPs.

Add the command and active response blocks to ossec.conf:

<command>

  <n>opnsense-block</n>

  <executable>opnsense-block.sh</executable>

  <expect>srcip</expect>

  <timeout_allowed>yes</timeout_allowed>

</command>

<active-response>

  <command>opnsense-block</command>

  <location>server</location>

  <rules_group>authentication_failures</rules_group>

  <level>10</level>

  <timeout>3600</timeout>

</active-response>

## **6.3 Response Action 2: Disable Local User Account**

When Wazuh detects a compromised or suspicious user account, this response disables the account on the affected host.

sudo nano /var/ossec/active-response/bin/disable-account.sh

#!/bin/bash

ACTION=$1

USER=$2

if [ "$ACTION" = 'add' ] && [ -n "$USER" ] && [ "$USER" != 'root' ]; then

  usermod -L "$USER"

  logger -t wazuh-ar "Disabled account: $USER"

fi

Add to ossec.conf:

<command>

  <n>disable-account</n>

  <executable>disable-account.sh</executable>

  <expect>user</expect>

</command>

<active-response>

  <command>disable-account</command>

  <location>local</location>

  <rules_group>authentication_failures</rules_group>

  <level>12</level>

</active-response>

## **6.4 Response Action 3: Restart Tampered Service**

When Wazuh detects that a monitored service binary has been modified via file integrity monitoring, this response restarts the affected service to restore a known-good running state while forensic collection runs.

sudo nano /var/ossec/active-response/bin/restart-service.sh

#!/bin/bash

SERVICE=$1

if [ -n "$SERVICE" ]; then

  systemctl restart "$SERVICE"

  logger -t wazuh-ar "Restarted service: $SERVICE"

fi

| **NOTE** | Service restart active response requires careful scoping. Apply it only to specific rule IDs related to binary modification, not broadly to all FIM alerts. A rule that fires on /etc/passwd modification should not restart sshd blindly. Review each rule ID before enabling this response. |
| --- | --- |

## **6.5 Response Action 4: Host Isolation via Proxmox API**

When a host is confirmed compromised at level 15, this response uses the Proxmox API to disconnect the VM's network interface, isolating it from the network while preserving the disk for forensic analysis.

sudo nano /var/ossec/active-response/bin/proxmox-isolate.sh

#!/bin/bash

AGENT_IP=$1

PVE_TOKEN=$(cat /etc/wazuh-proxmox/token)

PVE_HOST='https://10.0.0.13:8006'

# Lookup VMID from IP using Proxmox API

VMID=$(curl -sk -H "Authorization: PVEAPIToken=$PVE_TOKEN" \

  "$PVE_HOST/api2/json/cluster/resources?type=vm" \

  | python3 -c "

import json,sys

vms = json.load(sys.stdin)['data']

match = [v for v in vms if v.get('ip') == '$AGENT_IP']

print(match[0]['vmid'] if match else '')

")

if [ -n "$VMID" ]; then

  curl -sk -X PUT -H "Authorization: PVEAPIToken=$PVE_TOKEN" \

    "$PVE_HOST/api2/json/nodes/pve-env2/qemu/$VMID/config" \

    -d 'net0=virtio,bridge=vmbr0,link_down=1'

  logger -t wazuh-ar "Isolated VM $VMID (IP: $AGENT_IP)"

fi

| **WARNING** | Host isolation is a destructive action. Apply this response only to rule level 15 alerts after confirming the Proxmox API token has the minimum required permissions (VM.Config.Network only). Test in your lab environment against a disposable VM before enabling in production. |
| --- | --- |

## **6.6 Trigger Forensic Collection from Active Response**

Add a combined response that triggers Ansible forensic collection alongside any of the above responses. Create a wrapper script on wazuh-manager:

sudo nano /var/ossec/active-response/bin/ansible-forensics.sh

#!/bin/bash

AGENT_NAME=$1

CASE_ID=$2

ssh -i /etc/wazuh-ansible/id_ed25519 ryan@10.0.30.40 \

  ansible-playbook /opt/ansible/playbooks/forensic_collect.yml \

  -e "target_host=$AGENT_NAME case_id=$CASE_ID"

This script SSHes from wazuh-manager to VM 401 (ansible control node) and runs the forensic playbook against the affected host. The SSH key at /etc/wazuh-ansible/id_ed25519 must be authorized on VM 401.

# **7. Active Response Summary**

| **Response** | **Min Level** | **Trigger** | **Action** |
| --- | --- | --- | --- |
| OPNSense IP Block | 10 | authentication_failures group | Adds source IP to wazuh_blocked alias. Auto-expires after 1 hour. |
| Disable User Account | 12 | authentication_failures group | Locks the triggering user account on the affected host. |
| Restart Tampered Service | 12 | Specific FIM rule IDs | Restarts the service whose binary triggered the FIM alert. |
| Host Isolation | 15 | Manual trigger or critical malware rule | Disconnects VM network interface via Proxmox API. |
| Forensic Collection | 12 | Any high severity alert | Runs Ansible forensic playbook. Deposits output to soc-stack /opt/forensics. |

# **8. Verification**

| **Checkpoint** | **Status** |
| --- | --- |
| Ansible installed on VM 401 and all hosts return pong | **PASS / FAIL** |
| Ansible SSH keypair distributed to all managed hosts | **PASS / FAIL** |
| Forensic collection playbook runs and deposits output on soc-stack | **PASS / FAIL** |
| OPNSense IP block response tested against a known test IP | **PASS / FAIL** |
| User disable response tested in a controlled scenario | **PASS / FAIL** |
| Host isolation tested against a disposable VM | **PASS / FAIL** |
| All active response scripts have correct permissions (750, root:wazuh) | **PASS / FAIL** |
| Wazuh manager restarted and active response log shows no errors | **PASS / FAIL** |

# **9. Troubleshooting**

## **Active response script not executing**

Check permissions and ownership first:

ls -la /var/ossec/active-response/bin/

Scripts must be executable and owned by root:wazuh with permissions 750. Check the active response log:

sudo tail -f /var/ossec/logs/active-responses.log

## **Ansible playbook fails on SSH**

Test SSH connectivity from VM 401 to the target using the Ansible key:

ssh -i ~/.ssh/ansible_id ryan@<target_ip> 'whoami'

If this fails, the key was not distributed correctly. Re-run the distribution step for that host.

## **Forensic output not appearing on soc-stack**

Verify the fetch task in the playbook is using the correct destination path and that the ryan user on soc-stack has write permissions to /opt/forensics. Also check whether the staging directory on the target host was created successfully.

## **OPNSense API returning 401**

Verify the API key and secret are correct and that the API user in OPNSense has the Firewall Alias privilege assigned. Check that the API is enabled in System > Settings > Administration.


github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution
**Phase 4: Case Management with DFIR-IRIS**   |   April 19, 2026

**HOMELAB SOC**

Phase 4: Case Management with DFIR-IRIS

*Learning Exercise and Standard Operating Procedure*

| **Document Date** | April 19, 2026 |
| --- | --- |
| **Phase** | 4 of 5 |
| **Author** | Daniel Collins |
| **Prerequisite** | Phase 3 triage workflow established |
| **DFIR-IRIS Version** | v2.4.20 (latest stable) |
| **Deployment Host** | VM 200 (services-host, pve-env1) |
| **Estimated Time** | 3 to 5 hours (deploy and integrate) |

# **1. Purpose and Learning Objectives**

> **Status:** Planned
>
> DFIR-IRIS has not yet been deployed. This document defines the target architecture and deployment procedure for the case management integration.

---



A SIEM tells you something happened. A case management platform gives you the structure to investigate what happened, document your findings, track your actions, and close the loop. Without case management, incident response is ad hoc: analysts work in notes apps, emails, or memory. With case management, every investigation has a documented timeline that can be reviewed, audited, and learned from.

| **WHY** | Case management is how a SOC proves it investigated something. If you are ever in a position where you need to demonstrate due diligence during an audit, a regulatory review, or a post-incident debrief, your case records are the evidence. They show exactly what you knew, when you knew it, and what you did about it. |
| --- | --- |

## **1.1 About DFIR-IRIS**

DFIR-IRIS is a free and open source collaborative incident response platform originally built as a direct response to the licensing changes TheHive made in 2022. It is purpose-built for the incident response workflow and provides case management, evidence tracking, IOC management, timelines, task assignment, and reporting. It integrates with Wazuh via REST API and is deployed via Docker Compose.

DFIR-IRIS is used in production by SOC teams, MSSPs, and CERT organizations. Running it in this homelab demonstrates familiarity with the same tooling used in the industry.

## **1.2 Learning Objectives**

- Deploy DFIR-IRIS via Docker Compose on an existing Docker host

- Configure IRIS behind NPM for HTTPS access via internal DNS

- Understand the IRIS data model: cases, assets, IOCs, tasks, and timelines

- Create and manage a case manually through the full investigation lifecycle

- Write a Python script that creates IRIS cases automatically via the REST API when Wazuh generates high severity alerts

## **1.3 Success Criteria**

This phase is complete when DFIR-IRIS is accessible via HTTPS on the internal network, at least one case has been created manually and worked through to closure, and the Wazuh-to-IRIS automated case creation script is running and has created at least one case from a real alert.

# **2. Architecture**

## **2.1 Deployment Location**

DFIR-IRIS runs as a Docker Compose stack on VM 200 (services-host, pve-env1). This host already runs other Docker services including Vaultwarden, Uptime Kuma, Portainer, and Gitea. IRIS adds five Docker containers to that stack.

| **Container** | **Role** | **Notes** |
| --- | --- | --- |
| iris_webapp | Core app | Web server, API, database management, module handling |
| iris_worker | Task worker | Processes background jobs including module execution |
| iris_db | PostgreSQL | Persistent case data storage |
| iris_rabbitmq | Message queue | Task queue between webapp and worker |
| iris_nginx | Reverse proxy | Internal HTTPS, proxied behind NPM for external access |

## **2.2 Integration Model**

DFIR-IRIS does not receive data directly from Wazuh. The integration works through a Python script that runs on wazuh-manager (VM 601) and watches the Wazuh alert log. When the script detects a high severity alert, it calls the DFIR-IRIS REST API to create a case, populate it with alert details, and add the affected agent as a case asset.

This is an API-push model: the script pushes data into IRIS rather than IRIS pulling it. The REST API is the primary supported integration path for DFIR-IRIS and is fully maintained as part of the core platform.

# **3. Prerequisites**

- VM 200 (services-host) running with Docker and Docker Compose installed

- Portainer accessible for container management

- NPM running and able to proxy to VM 200

- Internal DNS entry available for iris.homelab.local (add via Pi-hole)

- Python 3 available on wazuh-manager (VM 601)

- Wazuh manager alert log accessible at /var/ossec/logs/alerts/alerts.json

# **4. Procedure: Deploy DFIR-IRIS**

## **4.1 Clone the Repository**

On VM 200 (services-host):

cd /opt/docker

git clone https://github.com/dfir-iris/iris-web.git iris

cd iris

git checkout v2.4.20

cp .env.model .env

## **4.2 Configure the Environment File**

Edit the .env file and set strong values for all passwords and secrets:

nano .env

Key values to set:

- POSTGRES_PASSWORD: set a strong password for the database

- IRIS_SECRET_KEY: set a long random string, used for session signing

- IRIS_SECURITY_PASSWORD_SALT: set another long random string

| **NOTE** | Generate strong values with: openssl rand -hex 32. Never use default values from the .env.model file in a running instance. |
| --- | --- |

## **4.3 Start the Stack**

docker compose pull

docker compose up -d

After the containers start, find the initial administrator password in the webapp container logs:

docker logs iris_webapp 2>&1 | grep 'post_init'

IRIS will be accessible at https://localhost:4433 on VM 200 initially.

## **4.4 Configure NPM Proxy**

In Nginx Proxy Manager, add a new proxy host:

- Domain: iris.homelab.local

- Forward hostname: VM 200 IP address

- Forward port: 4433

- Enable HTTPS with your internal certificate

Add the DNS entry in Pi-hole: iris.homelab.local pointing to the NPM IP.

## **4.5 First Login and Initial Configuration**

Browse to https://iris.homelab.local and log in with the administrator credentials found in the container log. Immediately change the administrator password under Profile settings.

Navigate to Advanced Settings and set the instance URL to https://iris.homelab.local. This ensures that links in API responses and notifications point to the correct address.

## **4.6 Generate an API Key**

The Wazuh integration script will authenticate to IRIS using an API key rather than username and password. To generate one:

- Log in as administrator

- Navigate to My Profile in the top right menu

- Scroll to API Key and click Generate

- Copy the key and store it in Vaultwarden under DFIR-IRIS API Key

| **NOTE** | The API key provides full access to IRIS as the account it belongs to. Store it securely and never hardcode it in scripts. The integration script will read it from an environment variable or a secrets file with restricted permissions. |
| --- | --- |

# **5. Procedure: Manual Case Workflow**

Before automating anything, work through a complete case manually. This builds familiarity with the IRIS data model and ensures you understand what the automation is producing.

## **5.1 Create a Case**

From the IRIS dashboard, click New Case. Fill in:

- Case name: use a consistent format such as CASE-001: [brief description]

- Description: paste the relevant Wazuh alert details

- Severity: match to Wazuh alert level (level 12+ maps to Critical, level 7 to 11 maps to High or Medium)

- TLP: use Green for internal homelab cases

## **5.2 Add Assets**

Assets are the hosts involved in the case. For each affected agent:

- Navigate to Assets in the left sidebar

- Click Add Asset

- Set the asset name to the agent hostname

- Set the asset type (Linux Server, Windows Server, Network Device, etc.)

- Add the IP address in the asset properties

## **5.3 Add IOCs**

Indicators of Compromise are specific artifacts from the alert: IP addresses, file hashes, usernames, or domains associated with the suspicious activity. For each IOC from the alert:

- Navigate to IOCs in the left sidebar

- Click Add IOC

- Set the IOC type (IP, hash, domain, username, etc.)

- Paste the value and add any relevant description

## **5.4 Build the Timeline**

The case timeline is a chronological record of events. Add timeline entries for:

- When the alert was generated (from the Wazuh @timestamp field)

- When you triaged the alert

- Each investigative action you took and what it revealed

- Resolution or escalation decision

## **5.5 Close the Case**

When investigation is complete, update the case status to Closed and add a summary note explaining the final determination. Closed cases remain in IRIS for future reference.

# **6. Procedure: Automated Case Creation Script**

## **6.1 Script Overview**

The integration script runs on wazuh-manager and watches /var/ossec/logs/alerts/alerts.json. When a new alert with a level of 12 or above appears, the script calls the DFIR-IRIS API to create a case.

## **6.2 Install Dependencies**

On wazuh-manager (VM 601):

sudo apt install python3-pip -y

pip3 install requests --break-system-packages

## **6.3 Create the Script**

sudo nano /opt/wazuh-iris-integration/iris_alert.py

Script contents:

#!/usr/bin/env python3

import json, sys, os, requests

from datetime import datetime

IRIS_URL  = os.environ.get('IRIS_URL', 'https://iris.homelab.local')

IRIS_KEY  = os.environ.get('IRIS_API_KEY')

MIN_LEVEL = int(os.environ.get('IRIS_MIN_LEVEL', '12'))

VERIFY_TLS = False  # set True once cert chain is trusted

HEADERS = {

    'Authorization': f'Bearer {IRIS_KEY}',

    'Content-Type': 'application/json',

}

def create_case(alert):

    rule  = alert.get('rule', {})

    agent = alert.get('agent', {})

    level = rule.get('level', 0)

    if level < MIN_LEVEL:

        return

    payload = {

        'case_name':  f"[AUTO] {rule.get('description','Unknown')} ({agent.get('name','unknown')})",

        'case_description': json.dumps(alert, indent=2),

        'case_severity':    1 if level >= 15 else 2 if level >= 12 else 3,

        'case_customer':    1,

        'case_classification': 1,

    }

    r = requests.post(f'{IRIS_URL}/api/v1/cases/add',

                      headers=HEADERS, json=payload, verify=VERIFY_TLS)

    r.raise_for_status()

    case_id = r.json()['data']['case_id']

    print(f'Created case {case_id} for alert level {level}')

if __name__ == '__main__':

    alert = json.loads(sys.stdin.read())

    create_case(alert)

## **6.4 Configure as a Wazuh Integration**

Wazuh has a native integration framework that calls external scripts when alerts fire. Add the following block to /var/ossec/etc/ossec.conf on wazuh-manager:

<integration>

  <name>custom-iris</name>

  <hook_url>https://iris.homelab.local</hook_url>

  <level>12</level>

  <alert_format>json</alert_format>

</integration>

Create the integration wrapper at /var/ossec/integrations/custom-iris:

#!/bin/bash

ALERT_FILE=$1

export IRIS_API_KEY=$(cat /etc/wazuh-iris/api.key)

export IRIS_URL='https://iris.homelab.local'

export IRIS_MIN_LEVEL=12

python3 /opt/wazuh-iris-integration/iris_alert.py < "$ALERT_FILE"

sudo chmod 750 /var/ossec/integrations/custom-iris

sudo chown root:wazuh /var/ossec/integrations/custom-iris

Store the API key in a restricted file:

sudo mkdir -p /etc/wazuh-iris

echo 'your-api-key-here' | sudo tee /etc/wazuh-iris/api.key

sudo chmod 640 /etc/wazuh-iris/api.key

sudo chown root:wazuh /etc/wazuh-iris/api.key

Restart wazuh-manager to load the integration:

sudo systemctl restart wazuh-manager

## **6.5 Test the Integration**

Trigger a test alert manually using the Wazuh logtest utility and verify a case appears in IRIS:

sudo /var/ossec/bin/ossec-logtest

Alternatively, watch the integration log for activity:

sudo tail -f /var/ossec/logs/integrations.log

# **7. Verification**

| **Checkpoint** | **Status** |
| --- | --- |
| DFIR-IRIS accessible at https://iris.homelab.local | **PASS / FAIL** |
| All five Docker containers running and healthy | **PASS / FAIL** |
| API key generated and stored securely in Vaultwarden | **PASS / FAIL** |
| At least one manual case created and closed with full documentation | **PASS / FAIL** |
| Integration script deployed and executable by wazuh user | **PASS / FAIL** |
| At least one automated case created from a real Wazuh alert | **PASS / FAIL** |


github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution
SOC Stack
The SOC environment runs on dedicated hardware isolated in VLAN10, separated from all production services, internal services, and the security lab. Nothing in VLAN10 has unsolicited inbound access from other VLANs. All components communicate over TLS using certificates issued by the internal PKI.
This is a working SOC stack, not a demo environment. Agents are deployed across the full cluster, logs are ingesting in real time, and dashboards are populated with live data.

Hardware
ComponentHostIPSpecsELK Stack (VM 600)pve-SOC (HP EliteDesk G6)10.0.10.10VLAN10Wazuh Manager (VM 601)pve-SOC (HP EliteDesk G6)10.0.10.118GB RAM, 2 cores, VLAN10
Both VMs run on the same dedicated physical node. pve-SOC is reserved exclusively for the SOC stack and runs nothing else.

Elastic Stack
Version: 8.19.12
Elasticsearch, Logstash, and Kibana run on VM 600 (soc-stack, 10.0.10.10).
TLS Configuration
TLS is enabled on both Elasticsearch and Kibana using certificates issued by the internal Intermediate CA. Kibana is accessible at https://elasticsearch.homelab.local via Nginx Proxy Manager. The Wazuh indexer connector points to Elasticsearch at https://10.0.10.10:9200 using internal PKI certificates.
DNS
soc-stack DNS resolves via Pi-hole on Heimdall. A DNS record for elasticsearch.homelab.local points to the NPM IP. OPNSense Unbound Query Forwarding routes homelab.local queries to the internal DNS server at 192.168.100.1.
Logstash Pipeline
Logstash runs a pipeline that ingests OPNSense firewall logs and forwards them to Elasticsearch. A drop filter is in place to suppress OPNSense log statistics noise messages that would otherwise pollute the index.
Active Indices
Fourteen wazuh-states-* indices are confirmed created and active. All index mappings are managed by the Wazuh indexer connector.

Wazuh
Version: 4.14.4
Wazuh Manager runs on VM 601 (wazuh-manager, 10.0.10.11). Filebeat is configured on the Wazuh Manager to ship alerts to Elasticsearch. Seventy-four documents were confirmed ingested on initial deployment.
Agent Deployment
Wazuh agents are deployed across all fourteen endpoints in the cluster using TLS enrollment. The agent rollout was performed sequentially across all targets.
TargetTypeNetworkservices-host (VM 200)Linux VMVLAN20kalshi-mm (VM 290)Linux VMVLAN20soc-stack (VM 600)Linux VMVLAN10wazuh-manager (VM 601)Linux VMVLAN10ubuntu (VM 401)Linux VMVLAN30pve-gatewayProxmox nodeVLAN1pve-servicesProxmox nodeVLAN1pve-env1Proxmox nodeVLAN1pve-env2Proxmox nodeVLAN1
OPNSense integration is configured for syslog forwarding. An agentless SSH check is also configured targeting the OPNSense management interface.
Docker Monitoring
The Wazuh Docker wodle is enabled on VM 200 (services-host). The wazuh user is added to the docker group on that host to allow container event collection without running the agent as root.

Kibana Dashboards
Seven official Wazuh dashboards are imported and populated in Kibana. All dashboards are confirmed live and rendering against real agent data.
DashboardPurposeSecurity EventsReal-time event stream across all agentsMalware DetectionFile integrity and malware alert correlationIncident ResponseAlert triage and incident trackingPCI-DSSCompliance monitoring against PCI-DSS controlsVulnerability ManagementCVE tracking across monitored endpointsDocker ListenerContainer event monitoring from services-hostOPNSense FirewallFirewall log ingestion and traffic analysis

Firewall Rules
The following OPNSense rules are in place to support SOC stack operation:
RuleProtocolPortPurposeSOC DNS (TCP)TCP53DNS resolution for SOC VLANSOC DNS (UDP)UDP53DNS resolution for SOC VLANLAN SSH to wazuh-managerTCP22Agent management access

Pending Items
ItemStatusOPNSense agentless SSH checkFailing — timeout to root@10.0.0.1Standard-PC-Q35-ICH9-2009 hostnameNoisy hostname needs hostnamectl fixVM 700 agent deploymentDeferred — VM was shut down at rollout timeLogstash ssl_verification_modeNot yet set to fullSuricata IPS modeCurrently in detection-only (IDS) modeRoot SSH login on Proxmox nodesNot yet disabled

Related Documentation

Network Architecture
Internal PKI
SOP: Security Hardening

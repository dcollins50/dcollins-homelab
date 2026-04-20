# SOC Stack

The SOC environment runs on dedicated hardware isolated in VLAN10, separated from all production services, internal services, and the security lab. Nothing in VLAN10 has unsolicited inbound access from other VLANs. All components communicate over TLS using certificates issued by the internal PKI.

This is a working SOC stack, not a demo environment. Agents are deployed across the full cluster, logs are ingesting in real time, and dashboards are populated with live data.

---

## Hardware

| Component | Host | IP | Specs |
|-----------|------|----|-------|
| ELK Stack (VM 600) | pve-SOC (HP EliteDesk G6) | 10.0.10.10 | VLAN10 |
| Wazuh Manager (VM 601) | pve-SOC (HP EliteDesk G6) | 10.0.10.11 | 8GB RAM, 2 cores, VLAN10 |

Both VMs run on the same dedicated physical node. pve-SOC is reserved exclusively for the SOC stack and runs nothing else.

---

## Elastic Stack

**Version:** 8.19.12

Elasticsearch, Logstash, and Kibana run on VM 600 (soc-stack, 10.0.10.10).

### TLS Configuration

TLS is enabled on both Elasticsearch and Kibana using certificates issued by the internal Intermediate CA. Kibana is accessible at `https://elasticsearch.homelab.local` via Nginx Proxy Manager. The Wazuh indexer connector points to Elasticsearch at `https://10.0.10.10:9200` using internal PKI certificates.

### DNS

soc-stack DNS resolves via Pi-hole on Heimdall. A DNS record for `elasticsearch.homelab.local` points to the NPM IP. OPNSense Unbound Query Forwarding routes `homelab.local` queries to the internal DNS server at `192.168.100.1`.

### Logstash Pipeline

Logstash runs a pipeline that ingests OPNSense firewall logs and forwards them to Elasticsearch. A drop filter is in place to suppress OPNSense log statistics noise messages that would otherwise pollute the index.

### Active Indices

Fourteen `wazuh-states-*` indices are confirmed created and active. All index mappings are managed by the Wazuh indexer connector.

---

## Wazuh

**Version:** 4.14.4

Wazuh Manager runs on VM 601 (wazuh-manager, 10.0.10.11). Filebeat is configured on the Wazuh Manager to ship alerts to Elasticsearch. Seventy-four documents were confirmed ingested on initial deployment.

### Agent Deployment

Wazuh agents are deployed across all fourteen endpoints in the cluster using TLS enrollment. The agent rollout was performed sequentially across all targets.

| Target | Type | Network |
|--------|------|---------|
| services-host (VM 200) | Linux VM | VLAN20 |
| kalshi-mm (VM 290) | Linux VM | VLAN20 |
| soc-stack (VM 600) | Linux VM | VLAN10 |
| wazuh-manager (VM 601) | Linux VM | VLAN10 |
| ubuntu (VM 401) | Linux VM | VLAN30 |
| pve-gateway | Proxmox node | VLAN1 |
| pve-services | Proxmox node | VLAN1 |
| pve-env1 | Proxmox node | VLAN1 |
| pve-env2 | Proxmox node | VLAN1 |

OPNSense integration is configured for syslog forwarding. An agentless SSH check is also configured targeting the OPNSense management interface.

### Docker Monitoring

The Wazuh Docker wodle is enabled on VM 200 (services-host). The `wazuh` user is added to the `docker` group on that host to allow container event collection without running the agent as root.

---

## Kibana Dashboards

Seven official Wazuh dashboards are imported and populated in Kibana. All dashboards are confirmed live and rendering against real agent data.

| Dashboard | Purpose |
|-----------|---------|
| Security Events | Real-time event stream across all agents |
| Malware Detection | File integrity and malware alert correlation |
| Incident Response | Alert triage and incident tracking |
| PCI-DSS | Compliance monitoring against PCI-DSS controls |
| Vulnerability Management | CVE tracking across monitored endpoints |
| Docker Listener | Container event monitoring from services-host |
| OPNSense Firewall | Firewall log ingestion and traffic analysis |

---

## Firewall Rules

The following OPNSense rules are in place to support SOC stack operation:

| Rule | Protocol | Port | Purpose |
|------|----------|------|---------|
| SOC DNS (TCP) | TCP | 53 | DNS resolution for SOC VLAN |
| SOC DNS (UDP) | UDP | 53 | DNS resolution for SOC VLAN |
| LAN SSH to wazuh-manager | TCP | 22 | Agent management access |

---

## Pending Items

| Item | Status |
|------|--------|
| OPNSense agentless SSH check | Resolved — `.passlist` entry was missing after manager restart; re-added `root@10.0.0.1 NOPASS`, confirmed `Test passed for 'ssh_integrity_check_bsd'` |
| Standard-PC-Q35-ICH9-2009 hostname | Noisy hostname needs `hostnamectl` fix |
| VM 700 agent deployment | Deferred — VM was shut down at rollout time |
| Logstash `ssl_verification_mode` | Not yet set to `full` |
| Suricata IPS mode | Currently in detection-only (IDS) mode |
| Root SSH login on Proxmox nodes | Not yet disabled |

---

## Related Documentation

- [Network Architecture](network.md)
- [Internal PKI](pki.md)
- [SOP: Security Hardening](sop-sec-001.md)

# SOC Stack

The SOC environment runs on dedicated hardware isolated in VLAN10, separated from all production services, internal services, and the security lab. Nothing in VLAN10 has unsolicited inbound access from other VLANs. All components communicate over TLS using certificates issued by the internal PKI.

This is a working SOC stack, not a demo environment. Agents are deployed across the cluster, logs are ingesting in real time, and dashboards are populated with live data. Beyond detection and log analysis, the stack now also includes alert automation (Shuffle SOAR) and incident case management (DFIR-IRIS).

---

## Hardware

| Component | Host | IP | Specs |
|-----------|------|----|-------|
| ELK Stack (VM 600) | pve-env2 (HP EliteDesk G6) | 10.0.10.10 | 16GB RAM, 8 cores, 200GB boot disk, 250GB NVMe passthrough, VLAN10 |
| Wazuh Manager (VM 601) | pve-env2 (HP EliteDesk G6) | 10.0.10.11 | 8GB RAM, 2 cores, VLAN10 |
| soar-host (VM 602) | pve-env1 (HP EliteDesk G5) | 10.0.10.12 | Shuffle SOAR |
| pve-iris (VM 604) | pve-env1 (HP EliteDesk G5) | 10.0.10.13 | DFIR-IRIS |

pve-env2 is reserved exclusively for the core ELK/Wazuh stack; soar-host and pve-iris run on pve-env1 since pve-env2's remaining capacity is reserved for additional agents.

---

## Elastic Stack

**Version:** 8.19.12

Elasticsearch, Logstash, and Kibana run on VM 600 (soc-stack, 10.0.10.10).

### TLS Configuration

TLS is enabled on both Elasticsearch and Kibana using certificates issued by the internal Intermediate CA. Kibana is accessible at `https://elasticsearch.homelab.local` via Nginx Proxy Manager. The Wazuh indexer connector points to Elasticsearch at `https://10.0.10.10:9200` using internal PKI certificates. Logstash's `ssl_verification_mode` is set to `full`.

### DNS

soc-stack DNS resolves via Pi-hole on Heimdall. A DNS record for `elasticsearch.homelab.local` points to the NPM IP. OPNSense Unbound Query Forwarding routes `homelab.local` queries to the internal DNS server at `192.168.100.1`.

### Logstash Pipeline

Logstash runs a pipeline that ingests OPNSense firewall logs and forwards them to Elasticsearch. A drop filter is in place to suppress OPNSense log statistics noise messages that would otherwise pollute the index. Heimdall's rsyslog output is also shipped to Logstash on a separate port from the main SOC ingest group.

### Log Retention

A 90-day retention policy applies to general Elasticsearch log indices. `wazuh-alerts` gets a longer 365-day window since it's lower-volume, higher-signal incident data; `wazuh-archives` stays on the 90-day window.

---

## Wazuh

**Version:** 4.14.4

Wazuh Manager runs on VM 601 (wazuh-manager, 10.0.10.11). Filebeat is configured on the Wazuh Manager to ship alerts to Elasticsearch.

### Agent Deployment

Wazuh agents are deployed across cluster endpoints via TLS enrollment, including:

| Target | Type | Network |
|--------|------|---------|
| services-host (VM 200) | Linux VM | VLAN20 |
| soc-stack (VM 600) | Linux VM | VLAN10 |
| wazuh-manager (VM 601) | Linux VM | VLAN10 |
| ubuntu (VM 401) | Linux VM | VLAN30 |
| kali-attack (VM 300) | Linux VM | VLAN40 |
| pve-gateway | Proxmox node | VLAN1 |
| pve-services | Proxmox node | VLAN1 |
| pve-env1 | Proxmox node | VLAN1 |
| pve-env2 | Proxmox node | VLAN1 |

Vulnerable lab targets (Metasploitable2, DVWA, malware-win11) do not run Wazuh agents given their intentionally compromised state.

OPNSense integration is configured for syslog forwarding. An agentless SSH check is also configured targeting the OPNSense management interface.

### Docker Monitoring

The Wazuh Docker wodle is enabled on VM 200 (services-host). The `wazuh` user is added to the `docker` group on that host to allow container event collection without running the agent as root.

---

## Kibana Dashboards

Seven official Wazuh dashboards are imported and populated in Kibana.

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

SOC net has general outbound access on HTTP/HTTPS (80/443). soc-stack has a dedicated WireGuard rule to Heimdall. DNS is restricted to OPNSense as the only permitted resolver; all other DNS destinations are blocked. LAN SSH is permitted to wazuh-manager only (TCP 22) for agent management, and wazuh-manager itself is permitted agentless SSH back to OPNSense. Wazuh agents on VLAN10 reach wazuh-manager on TCP/UDP 1514-1515. soar-host and pve-iris do not yet have dedicated firewall rules beyond the general SOC net outbound policy.

---

## Pending Items

| Item | Status |
|------|--------|
| Standard-PC-Q35-ICH9-2009 hostname | Noisy hostname needs `hostnamectl` fix |
| Suricata IPS mode | Currently in detection-only (IDS) mode |
| soar-host / pve-iris dedicated firewall rules | Not yet scoped, currently rely on general SOC net policy |

---

## Related Documentation

- [Network Architecture](network.md)
- [Internal PKI](pki.md)

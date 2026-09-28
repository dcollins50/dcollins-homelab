---
name: soc-stack-vm
type: vm
vmid: 600
node: pve-env2
ip: 10.0.10.10
vlan: VLAN10
status: live
role: Elasticsearch, Logstash, Kibana
aliases: [soc-stack]
---

**Login:** `admin`, SSH key auth (key copied Sep 25, 2026, confirmed with BatchMode test).

**Specs:** 16GB RAM, 8 cores, 200GB boot disk, 250GB NVMe passthrough (Elasticsearch data).
**Version:** Elastic Stack 8.19.12.

TLS enabled on Elasticsearch and Kibana via the internal Intermediate CA ([[pve-ca-intermediate]]). Kibana reachable at `https://elasticsearch.homelab.local` via internal NPM ([[services-host]]). Wazuh indexer connector points here at `https://10.0.10.10:9200`. Logstash `ssl_verification_mode` is `full` (hardened May 2, 2026).

DNS resolves via Pi-hole ([[heimdall]]); `elasticsearch.homelab.local` A record points to the NPM IP; OPNSense Unbound forwards `homelab.local` queries to 192.168.100.1.

**Logstash pipeline:** ingests OPNSense firewall logs, drop filter suppresses OPNSense stats-noise messages. Heimdall rsyslog ships on a separate port (5146) from the main SOC ingest group (5144).

**Retention:** 90 days for general log indices. `wazuh-alerts` gets 365 days (lower-volume, higher-signal). `wazuh-archives` stays on 90 days.

**Past incident:** hit `cluster.max_shards_per_node` (1000) default limit on Sep 13, 2026 — 938 indices/997 shards accumulated with no ILM policy to roll over/delete old daily indices. Root cause of the retention policy above.

## Kibana dashboards (7 imported)
Security Events, Malware Detection, Incident Response, PCI-DSS, Vulnerability Management, Docker Listener, OPNSense Firewall.

## Related
- [[pve-env2]]
- [[wazuh-manager]]
- [[pve-ca-intermediate]]
- [[VLAN10]]

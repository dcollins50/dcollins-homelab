**HOMELAB INFRASTRUCTURE**

Session Log & Implementation Record

March 30, 2026  |  SOC TLS Hardening, Wazuh Deployment, Firewall Remediation

| **Date** | March 30, 2026 |
| --- | --- |
| **Author** | Daniel Collins |
| **Affected Systems** | soc-stack (10.0.10.10), wazuh-manager (10.0.10.11), Heimdall, OPNSense, NPM, PKI |
| **Status** | Partial — core infrastructure complete, agent deployment pending |

# **Background**

This session continued infrastructure work from the March 28 session. Objectives were to fix soc-stack DNS, correct a stale Heimdall static route, and begin Wazuh deployment. The session expanded in scope to include TLS hardening across the ELK stack and remediation of additional firewall gaps discovered during implementation.

# **Work Completed**

## **1. soc-stack DNS Resolution Fix**

soc-stack was configured to use Pi-hole at 192.168.100.1 as its DNS resolver. OPNSense was blocking this traffic. The fix required enabling Unbound DNS on OPNSense to serve the SOC VLAN, adding a firewall rule permitting DNS from SOC net to OPNSense, and adding a Query Forwarding rule for homelab.local back to Pi-hole. soc-stack netplan was updated to point to 10.0.10.1.

- OPNSense SOC rule added: Allow SOC net → This Firewall → TCP/UDP 53

- Unbound Query Forwarding added: homelab.local → 192.168.100.1 (Pi-hole)

- soc-stack /etc/netplan updated: nameserver 192.168.100.1 → 10.0.10.1

## **2. Heimdall Stale Route Fix**

The route 10.99.0.0/24 via 192.168.100.2 was a stale temporary route pointing at a nonexistent device. It was present in the live routing table but absent from the NetworkManager persistent config, meaning it would have been lost on reboot anyway. The route was deleted and added correctly to the NM connection file.

- Deleted: sudo ip route del 10.99.0.0/24 via 192.168.100.2 dev eth0

- Added route2=10.99.0.0/24,192.168.100.250 to /etc/NetworkManager/system-connections/Wired connection 1.nmconnection

- Reloaded NM connection — route confirmed proto static metric 100

## **3. SSH Firewall Gap — LAN to Heimdall**

SSH to Heimdall from the workstation was discovered to be blocked. Sshd was active and listening on all interfaces. The gap was introduced during the February firewall hardening session — overly permissive rules were replaced with scoped rules but SSH to Heimdall was never explicitly added and never tested post-hardening. Tailscale (100.74.169.33) was used as out-of-band access to diagnose the issue.

- OPNSense LAN rule added: LAN net → 192.168.100.1 → TCP 22

- Root cause: firewall hardening session lacked end-to-end access validation

## **4. TLS on Elasticsearch and Kibana**

Elasticsearch and Kibana were configured to use TLS with certificates issued from the internal PKI. This required issuing certificates from the Intermediate CA, configuring Elasticsearch to serve HTTPS, updating Kibana to connect to Elasticsearch over HTTPS, and providing the intermediate CA cert to Kibana for chain validation.

A cert mismatch issue was encountered during reissuance — OpenSSL CA refused to reissue a cert with the same CN without first revoking the existing one. The cert was revoked and reissued with an IP SAN (10.0.10.10) added alongside the hostname SAN (elasticsearch.homelab.local). This was required for Filebeat on wazuh-manager to connect directly by IP.

- Cert issued: elasticsearch.homelab.local with SAN DNS=elasticsearch.homelab.local, IP=10.0.10.10

- Cert issued: kibana.homelab.local

- Elasticsearch xpack.security.http.ssl configured with PKI cert and key

- Kibana ssl configured with PKI cert, key, and intermediate CA as certificateAuthorities

- Kibana accessible at https://elasticsearch.homelab.local via NPM

- Pi-hole DNS: elasticsearch.homelab.local → NPM IP (10.0.20.30)

## **5. Logstash Pipeline Fix**

After Elasticsearch was restarted with TLS enabled, Logstash lost its connection. Additionally, a recurring grok timeout was identified — OPNSense was sending internal log statistics messages every 10 minutes that were too large for the grok filter to parse, blocking the pipeline worker. A drop filter was added to discard these messages before grok processing.

- Drop filter added to proxmox.conf: if [message] =~ 'Log statistics' { drop { } }

- Logstash restarted — pipeline healthy, all five index streams flowing

## **6. Wazuh Manager Deployment**

Wazuh Manager v4.14.4 was deployed as VM 601 on pve-SOC at 10.0.10.11. The VM was configured with 8GB RAM and 2 cores. Wazuh Manager-only deployment was chosen — no Wazuh Indexer or Dashboard, as the existing Elasticsearch stack handles those functions. Filebeat was installed on wazuh-manager to ship Wazuh alerts to Elasticsearch.

- VM 601 created: wazuh-manager, pve-SOC, 8GB RAM, 2 cores, 50GB disk, VLAN10

- Wazuh Manager v4.14.4 installed and running

- OPNSense LAN rule added: LAN net → 10.0.10.11 → TCP 22

- OPNSense SOC rule added: LAN net → 10.0.10.11 → TCP 22

- Filebeat 7.10.2 installed with Wazuh module

- Wazuh template and ingest pipeline loaded into Elasticsearch

- Filebeat connecting to https://10.0.10.10:9200 — wazuh-alerts-4.x index active with 74 initial documents

# **Issues Encountered**

## **Elasticsearch Cert/Key Mismatch**

During cert reissuance, the OpenSSL CA database prevented issuing a new cert with an existing CN without revocation. Multiple reissuance attempts resulted in the old cert being returned while a new key was generated, producing a mismatched pair. Resolution: revoke old cert before reissuance. Verification step added: always run modulus MD5 check on both cert and key before copying to destination.

## **Filebeat DNS/Proxy Routing Error**

Filebeat was initially pointed at elasticsearch.homelab.local which resolves to NPM at 10.0.20.30. NPM proxies port 443 only — it does not pass port 9200. Filebeat timed out connecting. The IP SAN was added to the Elasticsearch cert specifically to allow direct IP connections, but the hostname was used instead of the IP. Resolution: point Filebeat directly at 10.0.10.10:9200. NPM is for browser access only — backend service-to-service connections must use direct IPs or dedicated internal DNS records that resolve to the actual service host.

## **Kibana CA Trust Chain**

Kibana could not verify Elasticsearch's certificate because it did not trust the issuing CA. Both the intermediate CA cert and root CA cert were required in the certificateAuthorities array. Adding only the intermediate cert was insufficient.

# **Lessons Learned**

- **Always verify cert/key modulus match on the CA before copying to any destination**

- **Revoke existing certs before reissuance when changing SANs or extensions**

- **Backend service-to-service connections (Filebeat, Logstash) must connect directly — never route through a reverse proxy**

- **When adding IP SANs to a cert, use the IP for backend connections from the start**

- **Firewall hardening sessions must include end-to-end access validation before closing**

- **SCP multiple files in a single command: scp file1 file2 user@host:~/**

# **Pending Items**

| **Item** | **Notes** | **Status** |
| --- | --- | --- |
| Deploy Wazuh agents on VMs (200, 290, 401, 600, 700) | Standard Linux agent install | **Open** |
| Deploy Wazuh agents on Proxmox nodes (pve-gateway, services, env1, env2) | Agent on hypervisor host OS | **Open** |
| OPNSense os-wazuh-agent plugin | Install from Firmware → Plugins, configure Suricata EVE forwarding | **Open** |
| OPNSense agentless monitoring | SSH-based periodic config/integrity checks from Wazuh Manager | **Open** |
| Disable root password SSH on Proxmox nodes | Part of general hardening session | **Open** |
| Suricata tuning and IPS mode | After hardening session | **Open** |
| Logstash ssl_verification_mode => full | Currently set to none — update after CA cert deployed to Logstash | **Open** |
| soc-stack DNS fix | Changed to 10.0.10.1 via netplan | **Closed** |
| Heimdall stale route fix | 10.99.0.0/24 now via 192.168.100.250, persistent in NM | **Closed** |
| TLS on Elasticsearch and Kibana | Internal PKI certs, Kibana at elasticsearch.homelab.local | **Closed** |
| Wazuh Manager deployed | VM 601, 10.0.10.11, Filebeat shipping alerts to Elasticsearch | **Closed** |
| SSH firewall gaps | LAN SSH to Heimdall and wazuh-manager added | **Closed** |

github.com/dcollins50/dcollins-homelab  |  Internal Documentation  |  Not for Distribution
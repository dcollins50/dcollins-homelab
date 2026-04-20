# Internal PKI

This document covers the internal certificate authority hierarchy, certificate issuance procedures, and TLS deployment across the homelab environment. All internal services communicate over TLS using certificates issued by this PKI.

---

## Architecture Overview

The PKI uses a two-tier hierarchy: an offline Root CA and an online Intermediate CA. This is the same pattern used in production enterprise environments. The Root CA's private key is kept offline and is only brought online to sign the Intermediate CA certificate or issue a new one. Day-to-day certificate issuance is handled exclusively by the Intermediate CA.

```
Root CA (VM 500 — pve-services)
    |
    +— Intermediate CA (VM 501 — pve-services)
            |
            +— Elasticsearch
            +— Kibana
            +— Wazuh Manager
            +— Nginx Proxy Manager
            +— Proxmox nodes (all four)
            +— All internal services requiring TLS
```

The Root CA is planned to migrate from pve-services (VLAN30) to the management VLAN (VLAN1) for improved isolation.

---

## Certificate Authority VMs

| VM | ID | Host | Network | Role |
|----|----|------|---------|------|
| pve-ca-root | 500 | pve-services | VLAN30 | Root CA — offline except for CA operations |
| pve-ca-intermediate | 501 | pve-services | VLAN30 | Intermediate CA — issues leaf certificates |

---

## Root CA

The Root CA is the trust anchor for the entire environment. It signs only the Intermediate CA certificate. Its private key is not used for any other purpose.

**Operational posture:** The Root CA VM is shut down when not in use. It is only started to perform CA operations such as signing a new Intermediate CA certificate or renewing the existing one.

**Key algorithm:** RSA 4096 or Ed25519

**Trust distribution:** The Root CA certificate is installed in the trust store of all managed hosts and browsers that need to trust internal certificates. Any client that has the Root CA certificate in its trust store will automatically trust all certificates issued by the Intermediate CA.

---

## Intermediate CA

The Intermediate CA handles all day-to-day certificate issuance. It runs as a VM on pve-services and is online continuously to serve certificate requests.

**Signs:** Leaf certificates for all internal services

**Does not sign:** Other CA certificates

---

## Certificate Deployments

### Elastic Stack (VM 600)

TLS is enabled on both Elasticsearch and Kibana. Certificates are issued by the Intermediate CA and cover the following:

- Elasticsearch node-to-node transport encryption
- Elasticsearch HTTP API (port 9200) — used by Kibana, Logstash, and the Wazuh indexer connector
- Kibana HTTPS interface

The Wazuh indexer connector connects to Elasticsearch at `https://10.0.10.10:9200` using internal PKI certificates.

### Wazuh Manager (VM 601)

TLS is configured for:

- Wazuh agent enrollment (port 1515) via `wazuh-authd`
- Filebeat shipping alerts to Elasticsearch over HTTPS

The `sslmanager.cert` and `sslmanager.key` files are present on the Wazuh Manager for agent enrollment TLS.

### Nginx Proxy Manager

NPM terminates TLS for all internally proxied services using certificates issued by the Intermediate CA. All proxy hosts served through NPM are HTTPS-only.

### Proxmox Nodes

All four Proxmox nodes have internal PKI certificates deployed for their management web interfaces and API endpoints.

---

## Certificate Issuance Procedure

### Generating a Leaf Certificate

On the Intermediate CA VM:

```bash
# Generate a private key
openssl genrsa -out service.key 4096

# Generate a certificate signing request
openssl req -new -key service.key -out service.csr \
  -subj "/CN=service.homelab.local/O=Homelab"

# Sign the CSR with the Intermediate CA
openssl x509 -req -in service.csr \
  -CA intermediate-ca.crt -CAkey intermediate-ca.key \
  -CAcreateserial -out service.crt \
  -days 365 -sha256 \
  -extfile <(printf "subjectAltName=DNS:service.homelab.local,IP:10.x.x.x")
```

### Deploying to a Service

1. Copy `service.crt` and `service.key` to the target host
2. Copy `intermediate-ca.crt` and `root-ca.crt` as the CA chain
3. Configure the service to use the certificate and key paths
4. Restart the service
5. Verify TLS with `openssl s_client -connect host:port`

---

## Trust Store Distribution

For any client to trust internal certificates, the Root CA certificate must be added to its trust store.

**Linux (Debian/Ubuntu):**
```bash
sudo cp root-ca.crt /usr/local/share/ca-certificates/homelab-root-ca.crt
sudo update-ca-certificates
```

**Browser:** Import the Root CA certificate into the browser's certificate store under Trusted Root Certificate Authorities.

---

## Certificate Expiry Monitoring

Certificate expiry is monitored via Uptime Kuma running on services-host (VM 200). Uptime Kuma is configured to check TLS certificate validity for all internal HTTPS endpoints and alert when a certificate is approaching expiry.

---

## Backup and Recovery

| Item | Backup Location | Notes |
|------|----------------|-------|
| Root CA private key | Offline encrypted storage | Never stored on a network-connected device |
| Intermediate CA private key | Encrypted backup | Stored separately from the VM |
| Issued certificate inventory | Documented here | All deployments listed above |

If the Intermediate CA is lost, a new Intermediate CA can be created and signed by the Root CA. All leaf certificates will need to be reissued and redeployed to affected services. This is a significant operational event — the CA private key backup is critical.

---

## Planned Changes

| Item | Description |
|------|-------------|
| Root CA migration | Move Root CA VM from VLAN30 (pve-services) to VLAN1 (management) for improved isolation |
| Active Directory CS integration | Planned subordinate AD CS issuing CA under the existing offline Root CA. VM 501 will be decommissioned and replaced. Deferred until after Network+ exam. |

The AD CS integration will require reissuing certificates for all services currently under the Intermediate CA, including Elasticsearch, Kibana, Wazuh Manager, and NPM.

---

## Related Documentation

- [Network Architecture](network.md)
- [SOC Stack](soc-stack.md)
- [SOP: Security Hardening](sop-sec-001.md)

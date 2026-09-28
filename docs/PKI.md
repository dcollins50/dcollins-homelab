---
name: PKI
type: moc
status: live
---

Internal two-tier CA hierarchy and certificate issuance. All internal services communicate over TLS using certificates issued by this PKI. Host detail: [[pve-ca-root]], [[pve-ca-intermediate]], [[pve-int-stepca]].

## Architecture
```
Root CA (VM 500 — pve-services, VLAN30)
    |
    +— Intermediate CA (VM 501 — pve-services, VLAN30)
            |
            +— Elasticsearch
            +— Kibana
            +— Wazuh Manager
            +— Nginx Proxy Manager
            +— Proxmox nodes (all four)
            +— All internal services requiring TLS
```

Offline Root CA, online Intermediate CA — same pattern as production enterprise environments. Root CA's private key stays offline except to sign the Intermediate CA cert. Both CAs run on VLAN30 alongside [[authentik]] and [[pve-int-stepca]]. Identity/SSO runs through Authentik, not a Windows AD domain, so this PKI has no AD dependency.

## Certificate issuance procedure

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

This manual OpenSSL process will be superseded once the step-ca migration ([[pve-int-stepca]]) is validated and cut over.

**Deploying to a service:**
1. Copy `service.crt` and `service.key` to the target host
2. Copy `intermediate-ca.crt` and `root-ca.crt` as the CA chain
3. Configure the service to use the certificate and key paths
4. Restart the service
5. Verify TLS with `openssl s_client -connect host:port`

## Trust store distribution

**Linux (Debian/Ubuntu):**
```bash
sudo cp root-ca.crt /usr/local/share/ca-certificates/homelab-root-ca.crt
sudo update-ca-certificates
```

**Browser:** import the Root CA cert under Trusted Root Certificate Authorities.

## Certificate expiry monitoring
Monitored via Uptime Kuma on [[services-host]] — checks TLS validity for all internal HTTPS endpoints, alerts on approaching expiry.

## Backup and recovery
| Item | Backup Location | Notes |
|------|----------------|-------|
| Root CA private key | Offline encrypted storage | Never stored on a network-connected device |
| Intermediate CA private key | Encrypted backup | Stored separately from the VM |
| Issued certificate inventory | Documented across Hosts/ notes | — |

If the Intermediate CA is lost, a new one is signed by the Root CA and all leaf certs must be reissued/redeployed — a significant operational event, hence the private key backup being critical.

## Related
- [[pve-ca-root]]
- [[pve-ca-intermediate]]
- [[pve-int-stepca]]
- [[Network]]
- [[SOC-Stack]]

## SOPs
- [[sop-pki-migration-vlan30|PKI Migration to Trust Infrastructure VLAN]]
- [[sop-stepca-cutover|step-ca Cutover]]

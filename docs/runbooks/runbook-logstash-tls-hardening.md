# Runbook: Logstash TLS Certificate Verification Hardening

| Field | Value |
| --- | --- |
| Document ID | RUN-SOC-003 |
| Version | 1.0 |
| Status | Completed |
| Date Completed | May 2, 2026 |
| Author | Daniel Collins |
| Related SOP | SOP-SEC-001 Section 6.3 |

---

## Summary

Logstash was configured with `ssl_verification_mode => "none"` across all pipeline output blocks, meaning it would connect to Elasticsearch over TLS without verifying the server certificate. This was flagged as a known open item since the internal PKI was deployed. This runbook documents the steps taken to resolve it.

---

## Background

The homelab runs an internal two-tier PKI (Root CA on VM 500, Intermediate CA on VM 501, both on pve-services). All internal services including Elasticsearch use certificates issued by the Intermediate CA. Logstash on soc-stack (VM 600) was shipping logs to Elasticsearch but skipping certificate verification, leaving the connection vulnerable to interception on the internal network.

The Logstash pipelines affected were:

- `/etc/logstash/conf.d/proxmox.conf` -- 4 output blocks (proxmox, jetson, opnsense, heimdall)
- `/etc/logstash/conf.d/suricata.conf` -- 1 output block

---

## Root Cause

The internal CA certificate was never placed on soc-stack. Switching to `ssl_verification_mode => "full"` without it would have broken the pipeline immediately, so the setting was left as `"none"` as a temporary measure and never revisited.

---

## Steps Taken

### 1. Copy the Intermediate CA certificate to soc-stack

The Intermediate CA cert lives on VM 501 at `~/homelab-ca/intermediate-ca/certs/intermediate-ca.crt`. Direct SCP from soc-stack (VLAN10) to VM 501 (VLAN30) was blocked by firewall policy, so the workstation was used as a relay.

On the workstation:

```bash
scp ryan@10.0.0.21:~/homelab-ca/intermediate-ca/certs/intermediate-ca.crt .
scp intermediate-ca.crt ryan@10.0.10.10:/tmp/homelab-intermediate-ca.crt
```

On soc-stack:

```bash
sudo mkdir -p /etc/logstash/certs
sudo mv /tmp/homelab-intermediate-ca.crt /etc/logstash/certs/
sudo chown -R logstash:logstash /etc/logstash/certs
sudo chmod 640 /etc/logstash/certs/homelab-intermediate-ca.crt
```

### 2. Update Logstash pipeline configs

In every Elasticsearch output block in both pipeline files, replaced:

```ruby
ssl_verification_mode => "none"
```

with:

```ruby
ssl_verification_mode => "full"
ssl_certificate_authorities => ["/etc/logstash/certs/homelab-intermediate-ca.crt"]
```

Also updated all `hosts` entries from `https://localhost:9200` to `https://10.0.10.10:9200` to match the SANs on the Elasticsearch certificate (`elasticsearch.homelab.local`, `10.0.10.10`).

### 3. Validate config syntax

```bash
sudo -u logstash /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  --path.data /tmp/logstash-test \
  -f /etc/logstash/conf.d/proxmox.conf

sudo -u logstash /usr/share/logstash/bin/logstash \
  --config.test_and_exit \
  --path.data /tmp/logstash-test \
  -f /etc/logstash/conf.d/suricata.conf
```

Both returned `Configuration OK`.

### 4. Restart Logstash

The service restart stalled due to `TimeoutStopSec=infinity` in the systemd unit combined with inflight events that could not be delivered while Elasticsearch was unreachable. The process was force-killed and the service started manually:

```bash
sudo kill -9 <PID>
sudo systemctl start logstash
```

---

## Verification

Confirmed no certificate errors in `/var/log/logstash/logstash-plain.log` after restart.

Confirmed documents actively writing to Elasticsearch:

```bash
curl -s -u elastic:<password> https://10.0.10.10:9200/proxmox-logs-*/_count -k
```

Document count increased between successive calls. Today's index confirmed active:

```bash
curl -s -u elastic:<password> \
  "https://10.0.10.10:9200/proxmox-logs-$(date +%Y.%m.%d)/_count" -k
```

Returned a non-zero count confirming live writes.

---

## Notes

- The `.bak`, `.bak2`, and `.save` files in `/etc/logstash/conf.d/` are not loaded by the pipeline glob (`*.conf`) but should be cleaned up to reduce confusion.
- `TimeoutStopSec=infinity` in the systemd unit caused a slow shutdown when the pipeline had inflight events it could not flush. Consider setting a finite timeout (e.g., `TimeoutStopSec=60`) to avoid this in future restarts.
- The `suricata.conf` pipeline is active. Suricata 8.0.2 is running on OPNSense in IDS mode on the WAN interface with the Emerging Threats Open ruleset. EVE JSON is shipping to Logstash port 5147 and indexing into `suricata-logs-*` in Elasticsearch. TLS verification on that pipeline was also corrected in this session.

---

## Related Documentation

- [Internal PKI](../pki.md)
- [SOC Stack](../soc/soc-stack.md)
- [SOP-SEC-001 Security Hardening](../sop/sop-sec-001.md)

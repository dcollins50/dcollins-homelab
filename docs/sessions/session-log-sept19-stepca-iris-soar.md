# Session Log — September 19, 2026

**Scope:** A major deployment session covering four related pieces of work: standing up pve-int-stepca, deploying DFIR-IRIS, deploying Shuffle SOAR, and splitting the security lab VLANs. Documented here in roughly the order they were worked.

---

## 1. VLAN40/41 Security Lab Split

**Decision:** `malware-win11` (VM 400) is assigned to VLAN41 (fully air-gapped). `kali-attack`, `metasploitable2`, `dvwa`, and the Jetson Orin Nano are on VLAN40 (flat network, restricted outbound). This is the split that later got corrected into the current documentation, see [Security Lab](../../security-lab.md).

**Also decided:** VLAN51 is planned as a separate DMZ segment specifically to isolate the SSH Bastion from the rest of VLAN50 DMZ, rather than sharing a broadcast domain with public-facing services.

---

## 2. step-ca Migration Planning (pve-int-stepca)

**Decision:** Migrate the Intermediate CA from raw OpenSSL to step-ca by **importing the existing intermediate cert/key** rather than generating a fresh root, this preserves the existing trust chain rather than forcing every service to re-trust a new CA. Deployed on a separate new host, not in place on VM 501, so VM 501 stays as fallback until the new setup is validated.

**Placement decisions:**
- Ruled out a dedicated VM for step-ca.
- Decided on **Docker-in-LXC** over a plain LXC running step-ca as a native binary.
- Decided to build the LXC now, but defer step-ca's own configuration to a later dedicated session, when the plan is to migrate everything to step-ca at once rather than partially.
- Named the LXC **pve-int-stepca**, following the existing `pve-ca-root`/`pve-ca-intermediate` naming pattern.
- Placement: pve-services (grouped with Root CA, Intermediate CA, Authentik), on the Trust Infrastructure VLAN (30), same gateway as Authentik.
- Built from a Debian 13 template, consistent with every other LXC in this environment.

**Result:** pve-int-stepca (VMID 511) built and running on pve-services. Docker installed and confirmed working (`hello-world` ran successfully) via Docker-in-LXC with **nesting enabled**.

---

## 3. DFIR-IRIS Deployment

**Placement decision:** pve-env2 is capped out for the core SOC stack (remaining capacity reserved for additional Wazuh agents), so DFIR-IRIS was deployed as an LXC/VM on **pve-env1** instead, not pve-env2.

**Build:** pve-iris is a full clone of VM 603 (`docker-host.template`), assigned VMID 604, hostname `pve-iris`, on VLAN10 (SOC), sized 4 vCPU / 8GB RAM. Its IP had to be corrected to 10.0.10.13/24 after discovering the cloned template still carried soar-host's old IP (10.0.10.12), which would have conflicted with soar-host (VM 602) itself.

**Software:** IRIS v2.4.29, deployed via Docker Compose, fronted by NPM with a properly signed internal PKI cert (`iris.homelab.local`).

**Certificate issue hit during this work:** The SAN field was initially set incorrectly as `DNS:iris.homelab.local` instead of just `iris.homelab.local`, because the CA's config already prepends `DNS.` via `$ENV::SAN`, double-prefixing it. This required revoking the bad cert (serial 1013) and reissuing. **The correct pattern for this CA is `export SAN=iris.homelab.local`, no `DNS:` prefix**, before running `openssl ca`. Also worth noting for anyone working on this CA later: its config file is named `intermediate-ca.cnf`, not the more generic `openssl.cnf`.

**Firewall rules added:**
- Workstation → pve-iris (VLAN10)
- Internal NPM (Services, VLAN20) → pve-iris
- Workstation → soar-host

**Verification:** The IRIS API was confirmed working end-to-end via a PowerShell test call to `/alerts/add`.

**Design decision on alert-vs-case handling:** Automated triggers (from Shuffle) create IRIS **Alerts**, not **Cases**, deliberately, to avoid cluttering the actual case workload with false positives. A Case is for something a human has decided is worth actually investigating; an Alert is the raw automated signal.

---

## 4. Shuffle SOAR Deployment

**Build:** Deployed on soar-host (VM 602, pve-env1, 10.0.10.12). Docker installed, repo cloned, OpenSearch prerequisites completed. All six containers (frontend, backend, worker, orborus, security, opensearch) confirmed running.

**Firewall rule added:** Workstation to soar-host, ports 3001 and 3443, via a `soar_host` alias, to reach the web UI.

**Debounce workflow design (the actual point of standing this up):** Wait 60 seconds after Uptime Kuma's confirmed-DOWN event (this is *after* Kuma's own internal retries have already happened, not instead of them), then recheck. Only creates an IRIS ticket if the service is still down after that recheck. This directly serves the stated priority from earlier planning: reducing false-positive ticket generation, not generating a ticket on every transient blip.

**Recheck mechanism decision:** Use Kuma's `/metrics` Prometheus endpoint for the recheck rather than a direct HTTP ping, because it covers all of Kuma's monitor types uniformly (HTTP, TCP, etc.) instead of needing type-specific recheck logic.

**Integration issues hit during this work:**
- Kuma's webhook notification to Shuffle had to switch from HTTPS (port 3443) to HTTP (port 3001), because Kuma's underlying HTTP client (axios) rejects self-signed certs, and a separate firewall rule for port 3001 on the Services interface was needed as a result.
- Shuffle's webhook trigger node has to be **manually set to "Started"** and does not reliably persist that state across sessions, worth checking this specifically if a previously-working webhook stops firing.

**Workflow status at end of session:** Webhook trigger confirmed receiving real Kuma payloads (`heartbeat.status: 0` confirmed for a DOWN event). An "If else routing" condition node (`$exec.heartbeat.status = 0`) built and wired to the trigger. Still needed at end of session: the Wait/Delay (60s) node, the recheck step via Kuma's `/metrics`, and the conditional IRIS alert-creation node, this workflow was not yet complete end-to-end.

---

## Hardware context noted this session

pve-env1 specs: 6-core Intel i5-9500 @ 3.00GHz, 31.13 GiB RAM. Relevant since both soar-host and pve-iris landed here rather than on pve-env2, and this is the node whose headroom governed those placement decisions.

---

## Related Documentation

- [SOC Stack](../../soc-stack.md)
- [Internal PKI](../../pki.md)
- [Security Lab](../../security-lab.md)
- [Issue a Leaf Certificate](../runbooks/pki/issue-leaf-certificate.md)
- [Create an LXC](../runbooks/proxmox/create-lxc.md)

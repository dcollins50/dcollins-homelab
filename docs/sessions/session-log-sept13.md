# Runbook: Session Log — September 13, 2026

**Trigger:** Logs stopped shipping to Elastic following cluster restart after a 1–2 month shutdown.

**Scope:** What started as a single log-shipping investigation surfaced five separate, previously-undiscovered issues across Wazuh, Portainer, OPNSense/Pi-hole DNS, and Tailscale. Documented here in the order they were found and resolved.

---

## 1. Incident: Wazuh Alert Flood → Disk Full → Log Shipping Failure

### Symptom
Proxmox host logs were reaching Elastic; nothing else was. No obvious error on the ELK side.

### Investigation
- `systemctl status filebeat` on `wazuh-manager` showed the service `active (running)`, but the journal was flooded with:
  ```
  ERROR [registrar] registrar/registrar.go:205 Error writing registrar state to statestore:
  failed in store/get operation on store 'filebeat': write /var/lib/filebeat/registry/filebeat/checkpoint.new: no space left on device
  ```
- `df -h` confirmed `/` at 100% (96G/96G).
- `du -h --max-depth=1` traced the usage down through `/var` → `/var/ossec` → `/var/ossec/logs` → `/var/ossec/logs/alerts`, landing on two files:
  - `ossec-alerts-20.json` — 41G
  - `ossec-alerts-20.log` — 21G
  - Both from **August 20**, uncompressed, never rotated (every other day that month rotated normally at a few KB).

### Root Cause
- Sampling the file (`jq -r '.rule.id'` on the first 5,000 lines) showed **69% of alerts were rule 2501** ("syslog: User authentication failure"), all sourced from `wazuh-manager` itself with no `srcip` — internal, not an external attack.
- File timestamp (Aug 20, 14:34) lines up almost exactly with when the cluster is believed to have gone down. Working theory: something on the manager itself began failing auth repeatedly right around shutdown, and because the file was mid-write when the box went down, it never got the chance to rotate.
- Filebeat couldn't write its registry checkpoint once the disk hit 100%, so it stopped reliably shipping anything, not just the flooded alerts.

### Fix
```bash
# Both work with 0% free disk since they deallocate blocks in place
# rather than requiring space to write a new file elsewhere.
truncate -s 0 /var/ossec/logs/alerts/2026/Aug/ossec-alerts-20.json
truncate -s 0 /var/ossec/logs/alerts/2026/Aug/ossec-alerts-20.log

systemctl restart filebeat
```
Confirmed clean Filebeat startup: registry loaded, harvester started, connected to `https://10.0.10.10:9200`, index template loaded.

### Permanent Fix — Size-Capped Rotation
Added to `<global>` in `/var/ossec/etc/ossec.conf`:
```xml
<max_output_size>200M</max_output_size>
```
Restarted `wazuh-manager`. Rotation now triggers well before a runaway file can fill the disk (bounded by Wazuh's minimum 10-minute rotation-check interval).

Added a cleanup cron (Wazuh rotates/compresses automatically but never deletes):
```bash
0 0 * * * find /var/ossec/logs/alerts/ -type f -mtime +30 -exec rm -f {} \;
```

---

## 2. Portainer Agent — Docker Socket Exposed on Host Network

### Symptom
Routine post-incident check of `services-host` (`docker ps`) showed every container binding to `0.0.0.0` rather than a specific interface.

### Root Cause
`portainer-agent` had `-p 9001:9001` published, and its mounts included a bind of `/var/run/docker.sock` — effectively root-equivalent host access reachable from anywhere that could route to the container's port.

### Verification Before Fixing
- Confirmed via WAN firewall ruleset: **no port forwards exist**, so this was never internet-reachable — internal-network risk only.
- Confirmed `portainer-server` (172.20.0.10) and `portainer-agent` (172.20.0.11) share the `management-net` Docker network, so the published host port was unnecessary — Docker's internal DNS (`portainer-agent:9001`) is sufficient for the server to reach the agent.

### Fix
```bash
docker stop portainer-agent
docker rm portainer-agent
docker run -d \
  --name portainer-agent \
  --restart always \
  --network management-net \
  --ip 172.20.0.11 \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```
Updated the Portainer environment endpoint from `10.0.20.30:9001` (host IP) to `portainer-agent:9001` (internal DNS). Confirmed environment returned to `Up`.

---

## 3. Firewall Rule Scoping — Management ↔ Services

### Before
Both directions of `Services net` ↔ `LAN net` traffic used wildcard (any protocol, any port) rules.

### Method
Exported Uptime Kuma's full monitor list to determine *exactly* what `services-host` needs to reach across the VLAN boundary, rather than guessing. Findings:

| Target | Path | Already covered by |
|---|---|---|
| Proxmox nodes (`10.0.0.x:8006`) | Services → LAN | Existing `Proxmox_Nodes` alias rule |
| Vaultwarden/Portainer/Stremio/kalshi-mm | intra-VLAN (`10.0.20.x`) | N/A — never crosses the firewall |
| Google/Cloudflare DNS | Services → Internet | Existing internet-egress rule |
| OPNSense (`10.0.0.1`) | Services → gateway | Existing "Services to gateway" rule |
| **Pi-hole** (`192.168.100.1`, HTTP) | Services → LAN | **Nothing — needed a new rule** |
| **Heimdall** (`192.168.100.1`, ping) | Services → LAN | **Nothing — needed a new rule** |

### New Rules
**Management → Services:** `LAN net` → `10.0.20.30`, TCP, destination port = new alias `services_host_admin_ports`:
```
22, 80, 81, 443, 3000, 222, 9443, 8080, 3001
```
(derived from `ss -tulpn` on `services-host`: SSH, NPM 80/81/443, Gitea web 3000 + SSH 222, Portainer 9443, Vaultwarden 8080, Uptime Kuma 3001)

**Services → Management:** Split into two rules (ICMP and TCP can't share one OPNSense rule), both scoped to `192.168.100.1` only:
- TCP port 80 (Pi-hole)
- ICMP Echo Request (Heimdall)

### Gotcha
New rules initially sat below the pre-existing "Block Services to internal VLANs" rule (RFC1918 block), causing 100% packet loss on the ping test. OPNSense evaluates rules top-down, first match wins — reordered both new rules above the block rule to fix.

---

## 4. Pi-hole ⇄ Unbound DNS Forwarding Loop

### Symptom
Any `*.homelab.local` hostname not already in Pi-hole's static hosts list returned a raw network error on lookup instead of a clean NXDOMAIN — observed first as a `sudo: unable to resolve host` warning on `pve-env1`, later blocking `apt` entirely on that node.

### Root Cause
Genuine circular forward:
1. Pi-hole's `revServers` config: `true,10.0.0.0/24,10.0.0.1,homelab.local` — forwards **all** `homelab.local` queries (not just reverse/PTR) to OPNSense.
2. OPNSense's Unbound had its own Query Forwarding rule sending `homelab.local` queries straight back to Pi-hole (`192.168.100.1`).
3. Anything not already cached/hosted locally on either side bounced indefinitely.

### Fix
Removed/disabled the `homelab.local` domain association in OPNSense: **Services > Unbound DNS > Query Forwarding**. Pi-hole remains the source of truth for `homelab.local` hostnames; OPNSense no longer intercepts forward queries for that domain.

---

## 5. Fleet-Wide 15–20 Second `sudo` / Hostname Delay

### Symptom
`sudo whoami`, `hostname -f`, and similar took 15–33 seconds on `pve-env1` — plain `dig` (A record) lookups on the same hostname were instant (~1ms).

### Investigation
- `dig AAAA pve-env1.homelab.local` returned `communications error: timed out` — first against Tailscale's MagicDNS stub (`100.100.100.100`), then against Pi-hole directly (`192.168.100.1`) after disabling Tailscale's DNS override (`tailscale set --accept-dns=false`).
- Root cause was the **same loop-adjacent config from Issue #4**: Pi-hole's domain-wide conditional forward for `homelab.local` was sending AAAA queries to OPNSense's Unbound, which was hanging (not cleanly refusing) rather than replying NODATA — because `.local` is technically reserved for mDNS.
- `sudo` and `hostname -f` use `getaddrinfo()`, which queries both A and AAAA records; the hanging AAAA lookup was the entire delay. Plain `dig` in earlier tests only ever asked for A records, which is why it looked fast in isolation.

### Fix
Same as Issue #4 — clearing the domain association from Pi-hole's conditional forwarding entry fixed both problems simultaneously. `dig AAAA` now returns a clean `REFUSED` in ~6ms; `sudo whoami` dropped from 20–33s to ~16ms.

### Impact
This had almost certainly been silently costing 15–20 seconds on every `sudo` invocation and hostname lookup, on every `.homelab.local`-named host, for an unknown period prior to tonight — worth spot-checking other nodes if similar sluggishness has been noticed elsewhere.

---

## 6. Tailscale — Rediscovered, Repaired, Retained

### Finding
`tailscaled` was running on `services-host` with no prior documentation — described as a legacy break-glass remote access path from a previous incident.

### State at Discovery
7 machines on the tailnet (`desktop-m46kt80`, `heimdall`, `jetson`, `pve-dev`, `pve-gateway`, `pve-services`, `services-host`), all showing **node key expired since June 27, 2026** (~2.5 months). Confirmed via `tailscale ping` that the path was genuinely non-functional, not just displaying stale dashboard status.

### Secondary Bug — Re-Auth Hang
`sudo tailscale up` hung with no output on `services-host`. Root cause: Pi-hole was sinkholing `controlplane.tailscale.com` to `0.0.0.0` — a blocklist false positive. Fixed by whitelisting the domain in Pi-hole; confirmed via `dig controlplane.tailscale.com` returning real Tailscale IPs afterward.

### Resolution
- Re-authenticated all 7 nodes.
- Manually triggered client updates on outdated nodes via the Tailscale admin console (Owner/Admin/IT Admin role required).
- **Decision:** retain as a secondary break-glass path, backing up the Cloudflare Tunnel/Access admin path once fully built.
- **Open:** disable key expiry per-node (or via tailnet-wide policy) so this doesn't silently die again without anyone noticing.

---

## 7. MFA Enabled

- **Vaultwarden** — TOTP two-step login, recovery codes stored outside the vault.
- **Gitea** — TOTP two-factor, recovery codes stored outside Gitea. Note: web UI only — SSH access (port 222, key-based) is a separate auth path, unaffected.

---

## Summary Table

| Issue | Root Cause | Fix | Status |
|---|---|---|---|
| Log shipping failure | Disk full from unrotated alert flood | Truncated files, restarted Filebeat | Resolved |
| Recurrence risk | No size-based rotation | `max_output_size` + cleanup cron | Resolved |
| Portainer socket exposure | Unnecessary host port publish | Removed port, use internal DNS | Resolved |
| Overly broad firewall rules | Wildcard Management↔Services | Scoped to exact ports/hosts | Resolved |
| DNS loop | Pi-hole ↔ Unbound circular forward | Removed domain from conditional forwarding | Resolved |
| Fleet-wide sudo delay | Same DNS loop hanging on AAAA | Same fix as above | Resolved |
| Tailscale silently dead | Node key expiry, unmonitored | Re-authenticated, kept as backup path | Resolved; expiry policy still open |
| No MFA on password vault / git server | — | Enabled TOTP on both | Resolved |

# Session Log — September 21, 2026

**Scope:** Finish the step-ca migration (SSH cert OIDC provisioner), build the SSH Bastion's network segment and host, and get WARP Zero Trust device enrollment working end to end.

**Result:** step-ca fully migrated and OIDC-integrated with Authentik. DMZBastion (VLAN51) built with scoped firewall rules. pve-bastion LXC built, WARP client installed. A second Cloudflare Tunnel and LXC (pve-authtunnel) built specifically to publish Authentik for Cloudflare Access, since it had never been exposed publicly before. WARP device enrollment is configured correctly end to end but is currently blocked by a QUIC/HTTP3 protocol error on the final authorize redirect — carried over to next session.

---

## 1. step-ca: cert/key import and remote management

**Starting point:** pve-int-stepca (VMID 511) had Docker installed and confirmed working from a prior session, but step-ca itself had not been configured.

**Work done:**

- Discovered the existing intermediate cert's CN has a literal leading space (`CN=\ Homelab Intermediate CA`, confirmed via `openssl x509 -nameopt RFC2253`). Decided to use the correct value (no leading space) for the throwaway bootstrap CA name, and to defer fixing the real cert until cert rollout/revocation is automated.
- Ran `docker run` with `DOCKER_STEPCA_INIT_NAME` / `DOCKER_STEPCA_INIT_DNS_NAMES` to generate the boilerplate PKI (the "medium way" from Smallstep's own-CA tutorial).
- Copied the real root cert, intermediate cert, and intermediate key from pve-ca-intermediate onto pve-int-stepca via the Proxmox host (not directly — DMZ/VLAN30 has no route between them by firewall design), then `docker cp`'d them into the container, overwriting the throwaway files.
- Fixed a permissions error (`error reading intermediate_ca_key: permission denied`) — `docker cp` writes as root, but step-ca's process runs as UID 1000. Fixed with `chown 1000:1000`.
- Fixed a decryption error — the container's auto-generated `secrets/password` file (used to decrypt the CA's signing keys at boot) had to be overwritten with the *real* OpenSSL passphrase for the intermediate key, not left as the random value Docker generated.
- Verified the import by comparing SHA256 root fingerprints between step-ca's boot log and `pve-ca-root`'s actual cert — confirmed matching.

**Admin access / remote management:**

- `step ca provisioner add` requires the Admin API, which requires remote provisioner management to be enabled (off by default). Enabled it by editing `ca.json` inside the volume to add `"enableAdmin": true` and restarting, which auto-migrated the existing JWK provisioner and created a super admin (`step`).
- Lost the original auto-generated admin password when the password file was overwritten for the cert-import fix above. **Root-caused and fixed properly**: wiped the volume, rebuilt from scratch with `DOCKER_STEPCA_INIT_PASSWORD` and `DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true` set explicitly up front, and saved the resulting admin password immediately this time.

**Networking issues hit and fixed along the way:**

- `apt` hung on IPv6 attempts to `deb.debian.org` — root cause was Fastly (Debian's CDN backend) returning different edge IPs than what was cached; fixed by widening the `debian_repos` OPNSense alias from individual hostnames to the full set of Fastly's published IPv4 CIDR ranges (from `api.fastly.com/public-ip-list`).
- `curl` to `authentik.dcollinshomelab.org` failed cert verification from pve-int-stepca — the LXC's own trust store didn't have the homelab root CA. Fixed with `update-ca-certificates` after copying the root cert in (Debian, unlike the Alpine step-ca container, has proper tooling for this).

**OIDC provisioner:**

- Built a dedicated OAuth2/OIDC provider + application in Authentik named `cloudflare-access`... *(see section 3 — this ended up serving double duty)*. For step-ca specifically, the `Authentik` OIDC provisioner was added via:
  ```
  step ca provisioner add "Authentik" --type OIDC \
    --client-id <id> --client-secret <secret> \
    --configuration-endpoint https://authentik.dcollinshomelab.org/application/o/step-ca/.well-known/openid-configuration \
    --ssh --ca-url https://stepca.homelab.local:9000
  ```
- Confirmed via `step ca provisioner list` — registered correctly with `enableSSHCA: true`.

**Status:** Complete. step-ca is running on the real trust chain with a working Authentik OIDC provisioner for SSH cert issuance.

---

## 2. DMZBastion (VLAN51) network build

**Decision (this session):** SSH Bastion isolated onto its own VLAN (51), separate from general DMZ (VLAN50), so a compromise of the bastion has no default route to anything else, and vice versa.

**Switch config (TL-SG108E):**
- Added VLAN 51 ("DMZBastion") as tagged on Port 2 (pve-services) and Port 5 (OPNSense LAN).

**OPNSense config:**
- Created VLAN device `vlan09` (parent `re0`, tag 51).
- Assigned as interface `DMZBastion`, static `10.0.51.1/24`, no DHCP (static-only by design, matching the rest of the trust/security VLANs).

**Firewall rules (DMZBastion interface, all scoped to source alias `bastion` = `10.0.51.10`, not the whole `DMZBastion net`):**

| # | Destination | Port | Purpose |
|---|---|---|---|
| 1 | `10.0.51.1` | 53 | DNS to OPNSense's own resolver |
| 2 | `10.0.51.1` | 123 | NTP to OPNSense's own server |
| 3 | `cf_warp_ingress` (Cloudflare WARP IPv4 ranges) | `cf_warp_ports` (443, 500, 1701, 2408, 4443, 4500, 8095, 8443 UDP) | WARP connector tunnel |
| 4 | `cf_warp_api` (162.159.137.105, 162.159.138.105) | 443 | WARP client orchestration API |
| 5 | `cf_warp_doh` (162.159.36.1, 162.159.46.1) | 443 | WARP DNS-over-HTTPS |
| 6 | `internal_npm` | 443 | Authentik OIDC (for step-ca cert requests) |
| 7 | `pve_int_stepca` | 9000 | step-ca SSH cert issuance |
| 8 | `cf_edge_ipv4` (full Cloudflare edge range) | 443 | Package installs + general Cloudflare infra (subsumes #4/#5 but left separate for documentation clarity) |

- Also added a **temporary** bootstrap rule (`workstation` → `bastion`, port 22) for initial LXC setup, explicitly flagged to be removed once WARP enrollment is confirmed working, since a standing direct-LAN path would contradict the isolation goal of this VLAN.

**Status:** Complete.

---

## 3. pve-bastion LXC build

- Created VMID 2202, `pve-bastion`, Debian 13, **unprivileged**, on pve-services, static `10.0.51.10/24`, DNS pointed at `10.0.51.1` (not Heimdall — same mistake as pve-int-stepca avoided this time).
- Bootstrapping hit the same class of issues as pve-int-stepca (IPv6 hang on apt, `http://` vs `https://` sources.list mismatch against the firewall's port-443-only rule, Fastly IP range gap) — all fixed the same way.
- Installed `cloudflare-warp` package (official repo). This pulled in a large GUI dependency chain (GTK, WebKit, GStreamer, etc.) since the official Debian package bundles the full client, not just the headless daemon — noted as extra attack surface worth remembering, not fixed this session.
- Hit the known Debian 13 LXC `systemd-sysctl.service` failure (`243/CREDENTIALS`) — same root cause as any Debian 13 LXC without nesting (systemd's `ImportCreds=` doesn't work in unprivileged containers without it). Fixed by enabling `nesting=1` on the container.

**Status:** Host built, WARP client installed. Enrollment itself covered in section 5.

---

## 4. Publishing Authentik for Cloudflare Access (new tunnel + LXC)

**Root cause discovered:** Authentik has only ever been resolved *internally* (Pi-hole/Unbound overrides → `10.0.20.30`). It was never actually routed through the existing Cloudflare Tunnel (`web-tunnel`) to be reachable from the public internet — confirmed by checking `web-tunnel`'s Routes column (empty) and by a cellular-data test failing outright.

Cloudflare Access's own backend needs to reach the IdP's OIDC endpoints over the real public internet to do the token exchange — this is not something internal DNS tricks can work around.

**Decision:** rather than exposing Authentik directly, or publishing it through the existing general-purpose tunnel, built a **second, dedicated** Cloudflare Tunnel + LXC specifically for this, to keep the path that exposes the IdP isolated from the path that exposes public content.

- New LXC `pve-authtunnel`, VMID 2100, unprivileged, VLAN50 (DMZ — this is not part of the bastion's own isolation, it's a separate tunnel with no route to VLAN51), static `10.0.50.70/24`.
- New Cloudflare Tunnel `auth-tunnel`, `cloudflared` installed and registered as a systemd service.
- Firewall rules scoped narrowly: DNS/NTP to `10.0.50.1`, outbound to `cf_edge_ipv4` (tunnel + package installs), and a route to Authentik specifically (destination alias `authentik_host`, later corrected — see below).
- Public hostname route added: `authentik.dcollinshomelab.org` → origin.

**Cert trust chain issue:** first attempt pointed the route directly at Authentik's own listener (`10.0.30.10:9443`), which uses a **self-signed** cert (Authentik's own default), not one issued by the homelab CA. Result: Cloudflare edge returned Bad Gateway.

**Fix:** repointed the route at **NPM** (`10.0.20.30:443`) instead — the same path that already terminates TLS with a properly-issued cert, confirmed working when this was tested via curl earlier in the step-ca work. Updated both the Cloudflare route and the DMZ firewall rule (destination `authentik_host`/9443 → `internal_npm`/443).

**CA trust for the tunnel itself:** copied the homelab root cert onto pve-authtunnel (via the Proxmox host, same cross-VLAN-avoidance pattern as step-ca) and configured the route's **CA Pool** setting (`/etc/ssl/certs/homelab-root-ca.crt`) plus **Origin Server Name** (`authentik.dcollinshomelab.org`), rather than disabling TLS verification.

**Verified:** `https://authentik.dcollinshomelab.org/.well-known/openid-configuration` returns clean JSON from cellular data (fully outside the LAN) once these fixes landed.

**Status:** Complete. Authentik is now correctly and securely published to the public internet through its own dedicated tunnel.

---

## 5. Cloudflare Access + Authentik integration, WARP enrollment

- Built a second, separate Authentik OIDC provider/application (`cloudflare-access`, distinct from the `step-ca` one) specifically for Cloudflare Access itself.
- Added it as an OpenID Connect identity provider in the Zero Trust dashboard (Settings → Team & Resources → Integrations → Identity providers), using the Auth/Token/Certificate URLs pulled from Authentik's discovery document.
- **Root cause of most of the night's "Enrollment request is invalid" errors**, found after checking the OIDC identity provider test (which passed) and then digging further: **Device enrollment permissions had never been configured at all** — Cloudflare Zero Trust requires an explicit policy here before *any* device can enroll, separate from having an IdP integration configured. Fixed by creating an Allow policy (email-based) under Team & Resources → Devices → Management → Device enrollment permissions.
- After that fix, the WARP login screen correctly showed both `OIDC · Authentik` and the default `Cloudflare` OTP option — confirming the whole chain (tunnel → NPM → Authentik → Access → device enrollment policy) is wired correctly.

**Current blocker (unresolved, carried to next session):** the final step — clicking through to `/application/o/authorize/` — fails with `ERR_QUIC_PROTOCOL_ERROR` in the browser, reproduced across three different browsers (Safari/iOS, Chrome, Brave) and two networks (cellular, LAN). Server-side logs show the identification step succeeding cleanly (no error logged), suggesting the failure is in QUIC/HTTP3 protocol negotiation for this specific request rather than anything in the Authentik/Access/tunnel configuration. Leading theory: the request's `state` parameter is unusually large (several KB, base64-encoded), and QUIC has stricter size limits than HTTP/1.1 or HTTP/2 — worth confirming next session by testing the exact URL with `curl` (which uses HTTP/1.1/2) or by forcing HTTP/2 in the browser instead of disabling QUIC globally.

**Status:** Configured correctly, enrollment not yet completed. Next session should start with the curl test described above.

---

## Open items for next session

1. **Diagnose and fix the QUIC/authorize redirect failure** blocking WARP enrollment completion.
2. **Remove the temporary bootstrap SSH rule** (`workstation` → `bastion`, port 22) on DMZBastion once enrollment is confirmed working end to end.
3. **Reissue the real intermediate cert** to fix the leading-space CN typo — deferred until cert rollout/revocation is automated (per earlier decision, unchanged).
4. Wire sshd on pve-bastion to actually trust and request step-ca-issued SSH certs (the step-ca provisioner is ready; sshd-side configuration has not been started).

---
name: session-2026-10-03-bastion-warp-ipv6
type: session
date: 2026-10-03
---

Continued from [[session-2026-10-02-bastion-warp-enrollment]]. Goal: stop the WARP connect loop on [[pve-bastion]] without turning IPv6 back on for the whole container. Build plan is [[sop-ssh-bastion-build]].

## What was done
- Chose "IPv6 on the WARP interface only" over a Traffic only device profile. Reason: it changes nothing in Cloudflare, and `eth0` stays IPv4 only, so the `apt update` AAAA hang from [[proxmox-lxc-build-pattern]] cannot come back through `eth0`.
- Live test first, as root: set `net.ipv6.conf.default.disable_ipv6` to 0 at runtime with `all` and `eth0` left at 1. `warp-cli connect` then reported Connected with network healthy. `ip -br addr` showed `CloudflareWARP` with two global IPv6 addresses, one of them `2606:4700:cf1:2000::1`, the address the DNS proxy failed to bind on Oct 2. `eth0` and `lo` stayed IPv4 only and no other interface gained IPv6.
- Made it persistent: `default = 0` after the `all` line in both `/etc/sysctl.d/99-disable-ipv6.conf` and `/etc/sysctl.conf`. `/etc/sysctl.conf` is read last, so leaving `default = 1` there would undo the other file at boot.
- Rebooted the container with `pct reboot 2202`. After the reboot `all` was 1, `default` was 0, `eth0` was 1, and `warp-cli status` showed Connected with network healthy without a manual connect. Details in [[cloudflare-warp-mesh-node-headless]].
- Checked exposure before persisting. See the firewall finding below.

## Finding: the bastion has no host firewall yet
- sshd listens on `*:22`, every address and both IP families.
- `table inet filter` exists but its input, forward and output chains are empty with `policy accept`.
- WARP's own `table inet cloudflare-warp` accepts everything arriving on the `CloudflareWARP` interface. WARP manages that table, so it should not be edited.
- The Oct 2 session listed "a host firewall on the bastion" as one of the controls for Mesh SSH. That control does not exist yet. IPv6 on the WARP interface does not change this, because the IPv4 Mesh address was already accepted the same way.
- Today the controls over the Mesh path are the Cloudflare Mesh access policies (not yet checked) and sshd itself.

## Host firewall built (later the same day)
Daniel chose Mesh only for SSH (port 22 on `eth0` dropped) and outbound left open. Decision and rules: [[bastion-host-firewall]].
- `/etc/nftables.conf` on the bastion was the Debian default, starting with `flush ruleset`, and the nftables service is enabled. A `flush ruleset` in the boot file would wipe WARP's own table, so the new file has none and replaces only `table inet filter`. Old file backed up as `/etc/nftables.conf.bak-2026-10-03`.
- Order followed: write a candidate file, check it with `nft -c`, apply live only, confirm WARP still Connected, copy to `/etc/nftables.conf`, reboot, verify.
- After the reboot: `inet filter` and `inet cloudflare-warp` both present, input chain default drop with five rules, WARP Connected and healthy.
- Checked the service unit afterward: `ExecStop` runs `nft flush ruleset`, so stopping or restarting nftables on the bastion wipes WARP's table too. Use reload. See [[bastion-host-firewall]].

## Workstation enrolled, Mesh SSH tested (later the same day)
Daniel chose to install the client alongside Tailscale with a revert plan. Procedure and observations: [[cloudflare-warp-windows-workstation-mesh-access]].
- Tailscale baseline first. It started in `NoState` with the service Running, then connected. Before WARP: 11 `/32` Tailscale routes, `heimdall` pong in 7 ms direct.
- Installed and enrolled the Cloudflare One Client on the Windows 11 workstation. After connecting, the Tailscale routes and nine peers were unchanged. `jetson` and `services-host` sit inside the Mesh range but their `/32` routes still win.
- `ping 100.x.x.x` answered 3 of 3. Port 22 through the new firewall rule worked: `ssh [admin-user]@100.x.x.x` logged in and `SSH_CONNECTION` showed 100.x.x.x to 100.x.x.x.
- The host key was accepted at the first prompt before comparing. Compared afterward against the bastion's ED25519 key and it matched.
- `[admin-user]`'s password on the bastion was not remembered, so root set a new one with `passwd [admin-user]`. SSH is still password auth.

## Cloudflare Gateway policy review (read-only, later the same day)
Looked at the Cloudflare One dashboard (`https://dash.cloudflare.com/one/`), Traffic policies.
- Gateway default is allow: the overview says traffic and DNS are allowed unless a policy matches.
- Two policies exist, both created Sep 13, 2026: "Do Not Inspect" (HTTP) and "Default deny for private traffic" (Network).
- The default-deny policy is **disabled** (`enabled: false`). It is a Block at precedence 10000 on `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` and `100.96.0.0/12` (the Mesh range). There are no Allow policies. That is why ping and SSH to the bastion worked: nothing blocks Mesh traffic, so any enrolled device or user can reach any Mesh node on any port. The only gate today is the device enrollment permission.
- Traffic settings: "Allow Secure Web Gateway to proxy traffic" is on with TCP, UDP and ICMP all forwarded, so network policies would be enforced. TLS decryption is off (good for the later client certificate plan). Protocol detection is off.
- The overview showed 0 network sessions over 7 days despite Mesh pings and SSH. Cause not found: logging may be off or delayed.
- Plan agreed: create an Allow policy for the admin identity to 100.x.x.x on TCP 22 above the catch-all, test SSH, then enable the default-deny policy and test again.

## Cloudflare Gateway policies built and tested (later the same day)
Enrollment findings:
- Policy "Allow enrollment" (Allow, used by the application "Warp Login App"): one rule, Include, Emails, `user@example.com`. MFA is off at the policy level. Session duration follows the application session timeout (value not seen). Isolate application, purpose justification and manual approval are all off. "Accept all available identity providers" is on and "Apply instant authentication" is off.
- The login page for that application offers two methods: OIDC Authentik and Cloudflare. One-time PIN is not offered. The Identity providers list (Integrations, Identity providers) holds exactly two entries, Cloudflare and Authentik (OpenID Connect).
- The Cloudflare provider has "Restrict to account members" on. So that button only admits members of the Cloudflare account, and the enrollment rule still needs the email above. It is a login path that skips Authentik and Authentik's MFA. Kept on purpose, because it also works when Authentik is down. Revisit if a single login path is wanted.
- Devices page (Team and Resources, Devices): `[workstation]` registered as `user@example.com`, Default profile. `pve-bastion` registered as `warp_connector@[cf-team].cloudflareaccess.com`, Mesh Network Profile.
- Team domain mismatch, unexplained: the policy page shows `[cf-team].cloudflareaccess.com`, the login page header shows `[cf-team].cloudflareaccess.com`.

Network policies (Traffic controls, Firewall policies, Network tab), final state:
1. "Allow admin SSH to bastion": Allow. Destination IP is `100.x.x.x` AND Destination Port is `22`. Identity: User Email is `user@example.com`. Enforce session duration left on "Do not enforce". Active.
2. "Default deny for private traffic": Block on the RFC1918 ranges and `100.96.0.0/12`. Now **enabled**.
- The new Allow policy was created as number 2, below the deny policy. Gateway evaluates in order, so a deny at number 1 would have blocked the Allow. It was dragged above the deny policy (drag handle on the left of the row) before the deny policy was turned on.

Test results (workstation, new SSH connections to `100.x.x.x`, existing sessions prove nothing):
- Allow on, deny on: password prompt reached, login worked.
- Allow off, deny on: `kex_exchange_identification: read: Connection reset`, twice. Not a timeout.
- Allow back on: first attempt still reset (change had not propagated yet, under about a minute), second attempt reached the password prompt and logged in.
- So the Allow policy is what lets SSH through and the deny policy blocks the rest of TCP 22 on Mesh.
- Ping to `100.x.x.x` still worked with the deny policy on. Cloudflare's network policy docs describe TCP and UDP and do not address ICMP. Treat ICMP to Mesh nodes as not gated by these policies. The bastion's own firewall accepts ICMP anyway.
- The Matches (7 days) column stayed at a dash for both policies and network sessions stayed at 0 despite the tests. Cause not found.
- Not tested: ports other than 22, UDP, other users or devices, and whether the deny policy affects home LAN traffic if WARP split tunnel routes any of it (split tunnel list not checked, no breakage reported so far).

## Preparing step-ca SSH certificate login (later the same day)
- The `step` CLI was missing on the workstation. Installed with `winget install Smallstep.step` (version 0.31.0). A new PowerShell window is needed before `step` is found, because the old window keeps the old search path.
- From the workstation, `Test-NetConnection stepca.homelab.local -Port 9000` succeeded: the name resolves to `10.0.30.x` and the traffic leaves through the LAN adapter (`Ethernet 2`, source `10.0.0.x`), not the WARP interface. So the Gateway deny policy does not touch LAN traffic to step-ca.
- Vault check: `pve-int-stepca` (LXC 511 on `pve-services`) is `10.0.30.x`. `10.0.30.x` is `pve-ca-intermediate` (VM 501, old OpenSSL CA). Daniel briefly thought step-ca was `.22` and then corrected himself.
- On `pve-services`, `pct enter 511` needs `sudo` from the `admin` user. Inside the LXC the step-ca container is named `step-ca` (image `smallstep/step-ca`, port 9000, healthy). Running `step ssh config --roots` as `[admin-user]` on the LXC host fails with "requires the --ca-url flag", because the CLI there is not bootstrapped. The plan is to run it inside the container with `docker exec step-ca step ssh config --roots`. Not yet run.
- Still needed before sshd can trust step-ca: the SSH user CA public key, the root fingerprint for `step ca bootstrap` on the workstation, a real certificate to read the principal name from, and the Linux account choice. Smallstep's docs do not say what principal an OIDC certificate carries or how it maps to a Linux user, so read it off a real certificate with `ssh-keygen -L` instead of guessing.
- The step-ca OIDC provisioner is named "Authentik" and may share the Authentik application `cloudflare-access` with WARP enrollment. The Sep 20 session note flags that this could let the second login reuse the first session and skip a fresh MFA prompt. Not checked yet.

## What went wrong (for next time)
- My check for `flush ruleset` in the new boot file counted 1 because my own comment line contained those words. Use `grep -c "^flush ruleset"` so comments do not match.
- `route print -4 | findstr "100\."` also matches the gateway column, so it printed every WARP route once connected. It still showed what was needed, but narrow the filter next time.
- A stale paste of the previous step's output happened once again. Asking for the same command a second time cleared it.
- I told Daniel to use `sudo` on the bastion. It is not installed there. Writes need a root shell via `pct enter 2202` from [[pve-services]].
- I called the bastion's host firewall a control before looking at it. Look at `nft list ruleset` before relying on a firewall in a design.
- `sysctl` is not on a normal user's PATH. `grep -H . /proc/sys/net/ipv6/conf/<name>/disable_ipv6` works for reading values.

## Left open at the end of the session
- Whether Mesh peers can reach the bastion over its IPv6 addresses. Not tested.
- Mesh access policies are now done (see the Gateway policies section above). Still not done: sshd `TrustedUserCAKeys` for step-ca (SSH is still password auth), remove the temporary workstation to bastion port 22 rule on OPNsense.
- Unexplained: Matches column and network sessions stay at zero, and the team domain mismatch on the login page. ICMP to Mesh nodes is not gated by Gateway network policies.
- The earlier `ApiMismatch` and NTP log lines were not investigated. `warp-cli registration show` on Oct 3 showed a valid registration and no `ApiMismatch`. Check again only if the node drops.
- The workstation can now reach the bastion, so the admin portal design can continue. See [[admin-portal]].

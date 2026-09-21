# Runbook: Build a New Debian 13 LXC (Standard Pattern)

**Category:** Proxmox
**When to use:** Creating any new single-purpose LXC on this cluster. Covers the decisions and known first-boot issues that come up on essentially every fresh Debian 13 container here, so they don't get rediscovered from scratch each time.

## Decisions to make before creating

**Privileged vs. unprivileged.** Default to **unprivileged** unless the workload specifically needs low-level host device access (e.g. `/dev/net/tun` for a VPN client) that can't be granted safely any other way. Unprivileged is the safer default; only step down from it with a specific, named reason.

**Nesting.** Leave off by default. Turn on (`--features nesting=1`) if either:
- the workload runs Docker inside the LXC, or
- you hit the `systemd-sysctl.service` credentials bug described below (a known Debian 13 + unprivileged LXC issue, unrelated to Docker).

## Build

Standard fields for this cluster's convention:
- **Template:** Debian 13 standard
- **Hostname:** `pve-<purpose>`, matching the existing naming pattern
- **Network:** static IP on the appropriate VLAN, gateway matching that VLAN's OPNSense interface
- **DNS:** the VLAN's own OPNSense gateway address (e.g. `10.0.X.1`) — **not** Heimdall/Pi-hole unless this LXC's VLAN is specifically routed to reach it. Pointing a new LXC at Pi-hole from an isolated VLAN is the single most common first-boot networking mistake on this cluster; it works for internal `.homelab.local` names and then silently fails to resolve anything else, which looks like a firewall problem but isn't.
- **SSH key:** paste your public key at creation time to skip a manual step later:
  ```powershell
  Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
  ```

## First-boot fixes (check for these before assuming something else is wrong)

**1. `apt update` hangs, specifically on IPv6 addresses.** Debian's default mirror DNS often returns an AAAA record with no route from an isolated VLAN. Fix:
```bash
cat > /etc/sysctl.d/99-disable-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.eth0.disable_ipv6 = 1
EOF
sysctl --system
```
Verify after a reboot, not just after `sysctl -p` — the systemd-sysctl bug below can prevent this from actually applying at boot even though a manual apply looks like it worked.

**2. `apt update` fails on `http://` sources despite an apparently-correct firewall rule.** Debian 13's default `sources.list` uses `http://`, which a rule scoped to port 443 will correctly block. Fix by rewriting to `https://`, don't open port 80:
```bash
sed -i 's|http://deb.debian.org|https://deb.debian.org|g; s|http://security.debian.org|https://security.debian.org|g' /etc/apt/sources.list.d/debian.sources
```

**3. `systemd-sysctl.service` (and often other core services) fails with `243/CREDENTIALS`.** A known Debian 13 + unprivileged-LXC incompatibility with systemd's credentials-loading mechanism — not something wrong with your config.
```bash
# on the Proxmox host
pct set <vmid> --features nesting=1
pct stop <vmid> && pct start <vmid>
```
Confirm fixed: `systemctl status systemd-sysctl.service` should show `active (exited)` with no errors.

**4. Package mirror is CDN-backed and firewall rules keep almost-but-not-quite matching.** If the LXC's outbound access is scoped to specific destination aliases (rather than broad internet access) and `apt` intermittently times out even after fixes 1–2 above, the mirror's actual serving IP is likely rotating within a CDN provider's range (Fastly for Debian, Cloudflare for many others) faster than a narrow alias can track. Widen the destination alias to the provider's full published IPv4 range rather than individual resolved hostnames — see the OPNSense isolated-VLAN runbook for the general pattern.

## Verification

```bash
which curl        # confirm basic tooling present
apt update && apt upgrade -y     # should complete clean, no IPv6/http/CDN errors
```

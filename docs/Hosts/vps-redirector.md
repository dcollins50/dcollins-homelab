---
name: vps-redirector
type: physical-node
vmid: n/a
node: n/a
ip: <public-ip-redacted>
vlan: n/a (external / public internet)
status: live
role: Public-facing redirector for the security lab
---

**Login:** `admin`, sudo, key-only. Uses a dedicated ed25519 key (`vps_key`) kept separate from the lab SSH key, so a compromise of one does not expose the other. Root SSH login and password auth are both off.

External cloud VPS, so it has no Proxmox node and no homelab VLAN. a KVM VPS, Debian 13 64-bit, provider hostname `<vps-host>`, in the provider's Chicago location (node `<redacted>`). Specs: 512 MB RAM, 30 GB disk, 500 GB/month bandwidth. It is not part of the cluster and is reachable only over the public internet.

## Role: redirector, not an attack box

This box sits in front of the security lab so that engagement targets never see the home IP. Traffic aimed at a target hits this VPS's public address, and the VPS relays it back to [[kali-attack]] on [[VLAN40]], which lives behind [[opnsense]]. Two reasons for the design:

- **Hide the home IP.** Targets and any callbacks see `<public-ip-redacted>`, not the AT&T line the homelab rides on.
- **Contain a burned VPS.** If the redirector is compromised, the blast radius is this cheap throwaway box, not the homelab. The tunnel back home is meant to be scoped so a compromised VPS can only reach Kali's forwarded ports and nothing else in the lab or the rest of the network.

The actual offensive methodology (payloads, listeners, the attack chain itself) lives in the `pentest-notes` repo. This note covers the VPS only as standing infrastructure.

## Hardened state (Sep 28, 2026)

Base hardening is complete. Full procedure, generalized for any Debian VPS, is in [[sop-harden-debian-vps]]. What was done here:

- Non-root user `admin` with sudo, dedicated `vps_key` ed25519 key.
- Root SSH login and password auth disabled. This box shipped with a provider drop-in at `/etc/ssh/sshd_config.d/99-ctrl.conf` that forced `PermitRootLogin yes` and `PasswordAuthentication yes`, which overrode the main config. That file had to be removed, and the same two directives commented out in the main `sshd_config`, before the lockdown actually took effect.
- `nftables` default-deny inbound, allowing only SSH.
- `fail2ban` and `unattended-upgrades` installed.

Still planned: a Wazuh agent on this box reporting to [[wazuh-manager]] (10.0.10.11), so the redirector's own logs land in the SOC pipeline.

## Transport: currently Bore, moving to WireGuard

As built and validated, the VPS-to-Kali relay runs over **Bore**, not the intended WireGuard tunnel. Kali dials out to the VPS (home-initiated, which is the desired direction), but Bore traffic is cleartext and terminates on Kali itself, so it does **not** meet the encryption or containment goals above.

The plan is to replace Bore entirely with a **WireGuard tunnel terminating on [[opnsense]]**, home side initiating outbound, the VPS never dialing in. WireGuard also handles the NAT traversal that Bore was doing, so it is a straight replacement, not an addition. Containment will come from scoped OPNSense rules that limit the tunnel interface to Kali's forwarded ports only, with no allow into any other VLAN. That tunnel is not built yet; the phased build checklist is [[vps-wireguard-tunnel]], and it will get its own standing note once it exists.

## Teardown and reprovision

This box is meant to be burnable: rebuilt between engagements rather than nursed as long-lived state. The rebuild is script-based, not image-based.

**Source of truth:** the bootstrap script at `Scripts/bash/harden-debian-vps.sh` in the repo (gitignored, local only). It is idempotent and takes a fresh Debian box to the full hardened base in one run. Procedure and rationale: [[sop-harden-debian-vps]].

**Fast reprovision (the whole point):**
1. the provider panel: **Reinstall OS**, choose Debian 13. The box comes back bare (root + password). The reinstall is the only slow part, a few minutes provider-side.
2. Open the panel's **Console/VNC** as root (lockout-proof), or root-SSH in.
3. Put the script on the box (paste into the console, or `scp` it) and set `SSH_PUBKEY` to your `vps_key.pub` line.
4. `sudo bash harden-debian-vps.sh` (finishes in well under a minute).
5. Verify key login as `admin` works and root is refused. The box is back to hardened base.
6. Re-establish the transport as a separate step. The script does the base only, never the tunnel/relay. Today that means Bore; once the WireGuard tunnel exists it will get its own note and its own reprovision step.

**Lockout lifeboat.** If a hardening run ever locks you out (a bad nftables rule, an sshd typo), the panel's **Rescue Mode** boots the box into a recovery environment where you can mount the disk, fix the offending file, and reboot. That is a fuller recovery than Console/VNC alone, which still depends on the installed OS booting normally.

**No snapshot for this box.** the provider's control panel exposes no native snapshot or backup, only Reinstall OS (verified Sep 28, 2026). So reinstall plus the script *is* the reset path. If instant image-restore ever becomes worth it, a snapshot-capable provider (Vultr, already scoped as the Ansible-practice target: real snapshots, hourly billing, API) is the move, but the redirector stays on the provider for now. Note the panel also has an **API** tab: it cannot snapshot, but it is worth exploring whether the API can trigger the Reinstall itself, which would let the teardown step be scripted too, not just the post-reinstall hardening.

**One consistent drop-in name going forward.** The box was hardened by hand with the SSH policy in `99-hardening.conf`, which sorted *after* the provider's `99-ctrl.conf` and forced the manual removal of that file. The script standardizes on `00-hardening.conf`, which sorts before any provider drop-in and wins by first-match, so the removal is no longer load-bearing. After the next reprovision the box uses `00-hardening.conf`.

The only input the rebuild needs is the `vps_key` SSH public key. Keep it on the workstation. Nothing secret is baked into the baseline, which is what keeps the box safe to burn.

## Related
- [[Security-Lab]]
- [[kali-attack]]
- [[opnsense]]
- [[sop-harden-debian-vps]]

_Note: an earlier session spelled the IP `<public-ip-redacted>`; that was a typo. The confirmed address is `<public-ip-redacted>`._

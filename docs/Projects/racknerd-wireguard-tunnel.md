---
name: racknerd-wireguard-tunnel
type: project
status: in-progress
---

Build the encrypted transport for the security-lab redirector: a WireGuard tunnel between [[racknerd-vps]] and [[opnsense]] that replaces the current Bore relay. Targets hit the VPS public IP, the VPS relays across the tunnel, and OPNSense forwards to [[kali-attack]] on [[VLAN40]] while blocking everything else. Planned Sep 28, 2026. Nothing here is started. Values below (tunnel subnet, ports, NAT design) are proposed by Claude and pending Daniel's confirmation, not agreed, unless noted. Base VPS hardening is already done, see [[racknerd-vps]] and [[sop-harden-debian-vps]].

Why this replaces Bore: Bore is cleartext and terminates on Kali itself, so it meets neither the encryption goal nor the containment goal. WireGuard is encrypted, terminates on the firewall instead of the attack box, and handles the NAT traversal Bore was doing.

## Direction
OPNSense initiates outbound and holds the tunnel open with keepalives. The VPS never dials home and never has the home IP in its config. That gives NAT traversal (no inbound WAN rule at home) and auto-recovery if the home IP changes.

## Parameters to lock before running anything
- [ ] Tunnel subnet `10.31.74.0/30`: VPS `10.31.74.x`, OPNSense `10.31.74.x`. Confirm unused (soc-stack to Heimdall uses `10.31.73.x`).
- [ ] Kali target `10.99.0.x` on [[VLAN40]]. Confirm.
- [ ] Target-facing ports `4444` and `9000` (from the Bore setup). Confirm the real set.
- [ ] Design fork: full NAT on the VPS (Kali sees the tunnel as source, real target IP not visible at Kali) vs preserve real client IP (adds asymmetric-routing complexity). Default and config below assume full NAT.
- [ ] VPS public interface name (`ip -brief address` on the box). Called `WAN_IF` below.

## Phase 1: Keypairs
- [ ] VPS: `wg genkey | tee vps.key | wg pubkey > vps.pub`
- [ ] OPNSense: use the **Generate new keypair** button in the instance (Phase 2). Private keys never leave their own box; swap only the public keys.

## Phase 2: OPNSense (VPN then WireGuard)
- [ ] **Instances** tab, new instance: Enabled; Name `wg-racknerd`; Generate new keypair; Listen Port `51820`; MTU `1420`; Tunnel Address `10.31.74.x/30`.
- [ ] **Peers** tab, new peer: Enabled; Name `racknerd-vps`; Public Key = the VPS public key; Allowed IPs `10.31.74.x/32` only (this is the containment lever, the VPS can only ever present as this one address); Endpoint Address `[vps-public-ip]`; Endpoint Port `51820`; Keepalive `25`.
- [ ] Back on **Instances**, attach the peer to the instance.
- [ ] **VPN then WireGuard then Settings**: enable WireGuard.
- [ ] **Interfaces then Assignments**: assign the new `wgX` device as its own interface and enable it (needed so firewall rules can attach in Phase 6). Confirm the exact assignment screen on-device.
- [ ] Do NOT add an inbound WAN pass rule for UDP 51820 (home initiates). Verify outbound UDP 51820 to the VPS is not blocked (a past audit broke the soc-stack tunnel exactly this way).

## Phase 3: VPS WireGuard
- [ ] Write `/etc/wireguard/wg0.conf`:
```
[Interface]
Address = 10.31.74.x/30
ListenPort = 51820
PrivateKey = <contents of vps.key>

[Peer]
# OPNSense. No Endpoint line: home initiates.
PublicKey = <OPNSense instance public key>
AllowedIPs = 10.31.74.x/32, 10.99.0.x/32
```
- [ ] Enable it:
```
sudo apt-get install -y wireguard
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
sudo systemctl enable --now wg-quick@wg0
```

## Phase 4: Prove the tunnel (checkpoint, do not skip)
- [ ] `sudo wg` on both ends shows a recent handshake.
- [ ] VPS: `ping 10.31.74.x` reaches OPNSense.
- [ ] OPNSense (Diagnostics then Ping, sourced from the WireGuard interface): `ping 10.31.74.x`.
- [ ] If no handshake, open inbound UDP 51820 on the VPS (Phase 5 does this) and retest before debugging deeper.

## Phase 5: VPS firewall and NAT (nftables)
- [ ] Add to the **input** chain (currently SSH-only):
```
udp dport 51820 accept
tcp dport { 4444, 9000 } accept
```
- [ ] Add a NAT table (full NAT design; `WAN_IF` is the Phase-1 interface name):
```
table ip nat {
  chain prerouting  { type nat hook prerouting priority -100;
    iif "WAN_IF" tcp dport { 4444, 9000 } dnat to 10.99.0.x
  }
  chain postrouting { type nat hook postrouting priority 100;
    oif "wg0" ip daddr 10.99.0.x masquerade
  }
}
```
- [ ] Add to the **forward** chain (currently default drop):
```
ct state established,related accept
iif "WAN_IF" oif "wg0" ip daddr 10.99.0.x tcp dport { 4444, 9000 } accept
```

## Phase 6: OPNSense firewall on the tunnel interface
- [ ] Make an alias `racknerd_tunnel` = `10.31.74.x`.
- [ ] **Pass**, logged: proto TCP, source `racknerd_tunnel`, destination the `kali_attack` alias (`10.99.0.x`), ports `4444` and `9000`.
- [ ] **Block**, logged: source any, destination any. The explicit deny that proves containment.
- [ ] No NAT rule on OPNSense: this traffic is routed, not translated, on the home side. Kali's return traffic is allowed by state, so the existing "VLAN40 to internal VLANs" block is not touched.

## Phase 7: End-to-end test, including the deny
- [ ] From an outside host, connect to `[vps-public-ip]:4444` and confirm it lands on Kali's listener.
- [ ] Confirm the listener/reverse-shell catch works the same as it did over Bore.
- [ ] Deliberately try to reach something else across the tunnel (e.g. `10.0.10.x`) and confirm OPNSense drops and logs it. Test the block, not just the allow.

## Phase 8: Retire Bore and document
- [ ] Stop Bore on Kali and the VPS; remove the VPS firewall allow for Bore's control port.
- [ ] Trim the Bore specifics out of pentest-notes `PicoUSB-RevShell-build.md`.
- [ ] Write the real vault tunnel note and OPNSense-scoping doc (the rule was: only after it exists). Update [[racknerd-vps]] transport section from Bore to WireGuard. Add the transport step to `Scripts/bash/harden-debian-vps.sh` follow-on, or a separate tunnel-bringup script.

## Design fork detail
Full NAT on the VPS is simple and robust but Kali logs the tunnel address as the source, not the real attacker IP. Fine for a lab redirector, and the true source can still be captured at the VPS. Preserving the real client IP all the way to Kali means no source NAT plus policy-based return routing, which is the classic cause of asymmetric-routing problems. Start with full NAT; add client-IP preservation later only if actually needed.

## Related
- [[racknerd-vps]]
- [[kali-attack]]
- [[opnsense]]
- [[VLAN40]]
- [[sop-harden-debian-vps]]

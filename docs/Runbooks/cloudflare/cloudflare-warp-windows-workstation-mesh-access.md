---
name: cloudflare-warp-windows-workstation-mesh-access
type: runbook
tool: cloudflare
---

# Runbook: Windows workstation reaches a Mesh node (Cloudflare One Client)

**When to use:** A Windows machine must reach a Mesh node, here [[pve-bastion]] at 100.x.x.x. Built and tested Oct 3, 2026 on the Windows 11 workstation, which also runs Tailscale. Read [[cloudflare-warp-zero-trust-enrollment]] first for the IdP and device enrollment policy prerequisites.

**Status:** Verified on the workstation on Oct 3, 2026. Not tested: the phone, a workstation reboot with both clients installed, Gateway DNS policies.

## Steps
1. Get a Tailscale baseline first if Tailscale is installed (see the Tailscale section below).
2. Download the stable Windows installer from `https://downloads.cloudflareclient.com/v1/download/windows/ga` and run it. Approve the admin prompt.
3. Open the app, choose **Zero Trust security**, enter the team name `[cf-team]`, and finish the Authentik login in the browser, including MFA. On the success page, choose the option that opens the Cloudflare app.
4. The app starts **disconnected**. Click connect.
5. Verify from PowerShell: `ping -n 3 100.x.x.x`, then `ssh [admin-user]@100.x.x.x`.
6. The first SSH connection shows a host key fingerprint. Do not answer yes until it matches the bastion: on the bastion run `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` and compare the `SHA256:` string. It matched on Oct 3.
7. After login, `echo $SSH_CONNECTION` should show the workstation's Mesh address (100.x.x.x) first and 100.x.x.x third, which proves the path was the Mesh.

## What connected looks like
`route print -4` shows many routes through gateway 192.0.2.1 on the WARP interface. That is the default Exclude mode split tunnel: nearly everything goes through WARP and private ranges stay local. The block 100.64.0.0/10 is left out of the tunnel except 100.96.0.0/12, which is routed through WARP. That reading is from the route table, not from the dashboard.

## Tailscale on the same machine
- Cloudflare's Mesh tips page says Tailscale and similar software compete for control of routing and recommends removing them before deploying Mesh. That text is about running both on one machine.
- Observed here, no conflict: the 11 Tailscale `/32` host routes were unchanged after WARP connected, `tailscale status` still listed all nine peers, `tailscale ping heimdall` answered directly and `tailscale ping jetson` answered through the relay.
- Tailscale on Windows installs one `/32` route per peer, not a route for 100.64.0.0/10. Two peers sit inside the Mesh range: jetson (100.x.x.x) and services-host (100.x.x.x). Their `/32` routes are more specific than the Mesh `/12`, so they stay on Tailscale. If Mesh ever hands out an address equal to a Tailscale peer address, that Mesh node would be shadowed. That is an inference and was not tested.
- Oct 17, 2026: Tailscale's MagicDNS resolver `100.100.100.100` is inside the Mesh range 100.96.0.0/12, so its DNS traffic went into WARP and "Default deny for private traffic" blocked it (about every 10 minutes while Tailscale ran, 2,964 matches in a week, nothing visibly broken). Fixed with a split tunnel exclude `100.100.100.100/32` on the Default device profile. See [[session-2026-10-17-stepca-remote-tunnel]].
- Baseline commands before installing: `tailscale status` and `route print -4 | findstr "100\."`, then `tailscale ping -c 2 <peer>`.

## Quirks seen
- `tailscale status` first printed `unexpected state: NoState` with the service Running. This is a known Tailscale issue on Windows, see [Tailscale issue 15524](https://github.com/tailscale/tailscale/issues/15524). It connected a few minutes later after the Tailscale tray app was opened. Which step fixed it was not confirmed.
- `warp-cli` was not on the PATH in PowerShell after the install.
- `findstr "100\."` also matches the gateway column, so it prints every WARP route once WARP is connected. Narrow the filter when comparing.

## Undo
Switch WARP off in the app, or uninstall it from Settings, then Apps. Check `tailscale status` afterward.

## Unverified
- Whether Gateway DNS or network policies apply to the workstation beyond ICMP and SSH to the bastion.
- Which Mesh access policies exist, and whether other ports are blocked.

## Related Documentation
- [[cloudflare-warp-mesh-node-headless]]
- [[bastion-host-firewall]]
- [[tailscale-access-model]]
- [[session-2026-10-03-bastion-warp-ipv6]]

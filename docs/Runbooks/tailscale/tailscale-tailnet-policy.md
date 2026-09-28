---
name: tailscale-tailnet-policy
type: runbook
tool: tailscale
---

## Pi-hole sinkholing controlplane.tailscale.com

If `sudo tailscale up` hangs with no output and no error, before assuming a Tailscale-side problem, check whether Pi-hole is blocking Tailscale's own control-plane domain:
```bash
dig controlplane.tailscale.com
```
If this resolves to `0.0.0.0` (or doesn't resolve to a real Tailscale IP), Pi-hole's blocklists have almost certainly sinkholed it as a false positive, silently breaking every node's ability to re-authenticate or check in, with no visible error on the Tailscale side to point at DNS as the cause. Fix by whitelisting `controlplane.tailscale.com` in Pi-hole, then re-run `dig` to confirm it resolves to a real Tailscale IP before retrying `tailscale up`.

This previously caused every node on the tailnet to sit silently expired (as of a June 2026 key expiry) without anyone noticing, since nothing alerted on it, worth checking this domain specifically first any time Tailscale connectivity looks broken.

# Runbook: Apply the Tailnet Policy

**Status:** Applied Sep 23, 2026. Verified from the workstation (Proxmox UI on pve-env1 over Tailscale, Tailscale SSH to heimdall) and a denied path (services-host to pve-env1 port 22 blocked).
**Why:** The tailnet started on the default allow-all policy. See [[tailscale-access-model]] for the rules and reasoning.

## What it does
- Only the workstation is a source. All eight tailnet devices belong to one account, so a user-based source would have allowed node to node traffic.
- Workstation reaches SSH (22) and Proxmox (8006) on [[pve-env1]], [[pve-env2]], [[pve-gateway]], [[pve-services]].
- Workstation reaches SSH (22) only on [[services-host]], the jetson ([[jetson-orin-nano]]) and [[heimdall]].
- Everything else is denied. ICMP is allowed automatically between any pair with an allowed connection, so ping and traceroute work to every destination above.
- Tailscale SSH on [[heimdall]] needs both the network grant to port 22 and the `ssh` rule. The `ssh` rule uses action `accept`, chosen so break-glass does not depend on a periodic reauthentication check.
- The tests block makes Tailscale reject the policy if any assertion fails.

## Policy

```
{
  // Aliases pin the current tailnet addresses. Update them if you ever delete and re-enroll a node.
  "hosts": {
    "workstation":   "<tailscale-ip>",
    "heimdall":      "<tailscale-ip>",
    "jetson":        "<tailscale-ip>",
    "pve-env1":      "<tailscale-ip>",
    "pve-env2":      "<tailscale-ip>",
    "pve-gateway":   "<tailscale-ip>",
    "pve-services":  "<tailscale-ip>",
    "services-host": "<tailscale-ip>",
  },

  "grants": [
    { "src": ["workstation"],
      "dst": ["pve-env1", "pve-env2", "pve-gateway", "pve-services"],
      "ip":  ["tcp:22", "tcp:8006"] },
    { "src": ["workstation"],
      "dst": ["services-host", "jetson", "heimdall"],
      "ip":  ["tcp:22"] },
  ],

  "ssh": [
    { "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot"] },
  ],

  "tests": [
    { "src": "workstation",
      "accept": ["pve-env1:22", "pve-env1:8006", "pve-env2:22", "pve-env2:8006",
                 "pve-gateway:22", "pve-gateway:8006", "pve-services:22", "pve-services:8006",
                 "services-host:22", "jetson:22", "heimdall:22"],
      "deny":   ["services-host:8006", "jetson:8006", "heimdall:8006"] },
    { "src": "workstation", "proto": "icmp",
      "accept": ["pve-env1:0", "pve-services:0", "services-host:0", "jetson:0", "heimdall:0"] },
    { "src": "jetson",
      "deny": ["pve-env1:22", "pve-services:22", "pve-services:8006", "heimdall:22", "services-host:22"] },
    { "src": "services-host",
      "deny": ["pve-env1:22", "pve-env1:8006", "jetson:22", "heimdall:22"] },
    { "src": "pve-env1",
      "deny": ["pve-env2:22", "pve-services:22", "pve-services:8006"] },
  ],
}
```

## Apply
1. Be on the LAN, not relying on the tailnet, while you do this.
2. Open Access controls in the Tailscale admin console.
3. Replace the policy file contents with the block above and save. If a test fails, Tailscale rejects the save and names the failing test.
4. From the workstation over Tailscale, confirm SSH to each destination and the Proxmox web UI on 8006 for the four PVE nodes.
5. Confirm Tailscale SSH still works to [[heimdall]].

## Rollback
Policy changes are logged in the configuration audit logs and can be reverted from there. Tailscale's own reset option restores the original allow-all default.

## Not covered here
The bastion rules live outside Tailscale (OPNsense, no Tailscale client on the bastion, no route advertisement on [[pve-services]]). See [[tailscale-access-model]].

## Related
- [[Tailscale]]
- [[tailscale-access-model]]

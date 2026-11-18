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

**Status:** Applied Sep 23, 2026. Extended Sep 28 to 30 (dev-git, nimbus), Oct 23, 2026 ([admin-user]-worktop to nimbus), and at an unrecorded date between Oct 23 and Nov 7, 2026 (achilles share, pve-ansible, the Asscord staging VPS). Verified from the workstation (Proxmox UI on pve-env1 over Tailscale, Tailscale SSH to heimdall) and a denied path (services-host to pve-env1 port 22 blocked).
**Why:** The tailnet started on the default allow-all policy. See [[tailscale-access-model]] for the rules and reasoning.

## What it does
- The workstation is the main admin source. Other sources are each limited to named destinations and ports.
- Workstation reaches SSH (22) and Proxmox (8006) on [[pve-env1]], [[pve-env2]], [[pve-gateway]], [[pve-services]].
- Workstation reaches SSH (22) only on [[services-host]], the jetson ([[jetson-orin-nano]]) and [[heimdall]].
- [[dev-git]] and [[nimbus]]: see their grants below and [[gitea-collab-access]], [[cloud-files-design]].
- achilles: a machine shared in from the collaborator's tailnet. The workstation and [admin-user]-worktop reach its web page on port 80 only.
- Asscord staging VPS: a Hostinger server identified by the tag `tag:asscord-staging`, not by a host alias. The workstation and [[pve-ansible]] reach it on SSH (22) only. Nothing grants the VPS itself any access, so it is denied everything by default.
- [[pve-ansible]] reaches [[dev-git]] on Gitea SSH (2222) only, to clone `[admin-user]/asscord` with a read only deploy key.
- Everything else is denied. ICMP is allowed automatically between any pair with an allowed connection, so ping and traceroute work to every destination above.
- Tailscale SSH on [[heimdall]] needs both the network grant to port 22 and the `ssh` rule. The `ssh` rule uses action `accept`, chosen so break-glass does not depend on a periodic reauthentication check.
- The tests block makes Tailscale reject the policy if any assertion fails.

## Known gaps (Nov 7, 2026)
- No test proves the Asscord staging VPS cannot reach anything. Default deny covers it today, but a test would catch a future grant that opens it up.
- No test covers pve-ansible to the Asscord VPS, or pve-ansible to anything other than dev-git.

## Policy

Live policy as of Nov 7, 2026 (pasted by Daniel; replaces the Oct 23 copy, which lacked the achilles, pve-ansible and Asscord entries):

```
{
	"tagOwners": {
		"tag:asscord-staging": ["autogroup:admin"],
	},

	// Aliases pin the current tailnet addresses. Update them if you ever delete and re-enroll a node.
	"hosts": {
		"workstation":   "100.x.x.x",
		"heimdall":      "100.x.x.x",
		"jetson":        "100.x.x.x",
		"pve-env1":      "100.x.x.x",
		"pve-env2":      "100.x.x.x",
		"pve-gateway":   "100.x.x.x",
		"pve-services":  "100.x.x.x",
		"services-host": "100.x.x.x",
		"dev-git":       "100.x.x.x",
		"nimbus":        "100.x.x.x",
		"[admin-user]-worktop":  "100.x.x.x",
		"achilles":      "100.x.x.x",
		"pve-ansible":   "100.x.x.x",
	},

	"grants": [
		{
			"src": ["workstation"],
			"dst": ["pve-env1", "pve-env2", "pve-gateway", "pve-services"],
			"ip":  ["tcp:22", "tcp:8006"],
		},
		{
			"src": ["workstation"],
			"dst": ["services-host", "jetson", "heimdall"],
			"ip":  ["tcp:22"],
		},
		{
			// You manage dev-git: Gitea web, Gitea SSH
			"src": ["workstation"],
			"dst": ["dev-git"],
			"ip":  ["tcp:3000", "tcp:2222"],
		},
		{
			// Shared users (your friend) get Gitea web and Gitea SSH only
			"src": ["autogroup:shared"],
			"dst": ["dev-git"],
			"ip":  ["tcp:3000", "tcp:2222"],
		},
		{
			// You manage nimbus: web only, container SSH is off
			"src": ["workstation"],
			"dst": ["nimbus"],
			"ip":  ["tcp:80", "tcp:443"],
		},
		{
			// Shared users (the collaborator) get the nimbus web page only
			"src": ["autogroup:shared"],
			"dst": ["nimbus"],
			"ip":  ["tcp:80", "tcp:443"],
		},
		{
			// [admin-user]-worktop: nimbus web over HTTPS only
			"src": ["[admin-user]-worktop"],
			"dst": ["nimbus"],
			"ip":  ["tcp:443"],
		},
		{
			// achilles: shared in from the collaborator's tailnet, web only
			"src": ["workstation", "[admin-user]-worktop"],
			"dst": ["achilles"],
			"ip":  ["tcp:80"],
		},
		{
			// asscord staging VPS: SSH from the workstation and the Ansible control node only
			"src": ["workstation", "pve-ansible"],
			"dst": ["tag:asscord-staging"],
			"ip":  ["tcp:22"],
		},
		{
			// Asscord source: pve-ansible clones [admin-user]/asscord from dev-git (read-only deploy key).
			// Gitea SSH is on 2222, not 22.
			"src": ["pve-ansible"],
			"dst": ["dev-git"],
			"ip":  ["tcp:2222"],
		},
	],

	"ssh": [
		{
			"action": "accept",
			"src":    ["autogroup:member"],
			"dst":    ["autogroup:self"],
			"users":  ["autogroup:nonroot"],
		},
	],

	"tests": [
		{
			"src":    "workstation",
			"accept": ["nimbus:80", "nimbus:443"],
			"deny":   ["nimbus:22"],
		},
		{
			"src":    "[admin-user]-worktop",
			"accept": ["nimbus:443"],
			"deny":   ["nimbus:22", "nimbus:80", "pve-env1:22", "pve-env1:8006", "workstation:22"],
		},
		{
			"src":    "[admin-user]-worktop",
			"accept": ["achilles:80"],
			"deny":   ["achilles:22"],
		},
		{
			"src": "nimbus",
			"deny": [
				"pve-env1:22",
				"pve-env1:8006",
				"services-host:22",
				"heimdall:22",
				"workstation:22",
				"dev-git:3000",
				"dev-git:2222",
			],
		},
		{
			"src": "workstation",
			"accept": [
				"pve-env1:22",
				"pve-env1:8006",
				"pve-env2:22",
				"pve-env2:8006",
				"pve-gateway:22",
				"pve-gateway:8006",
				"pve-services:22",
				"pve-services:8006",
				"services-host:22",
				"jetson:22",
				"heimdall:22",
				"dev-git:3000",
				"dev-git:2222",
			],
			"deny": ["services-host:8006", "jetson:8006", "heimdall:8006", "dev-git:22"],
		},
		{
			"src":    "workstation",
			"proto":  "icmp",
			"accept": ["pve-env1:0", "pve-services:0", "services-host:0", "jetson:0", "heimdall:0"],
		},
		{
			"src":    "workstation",
			"accept": ["achilles:80"],
			"deny":   ["achilles:22"],
		},
		{
			"src": "jetson",
			"deny": [
				"pve-env1:22",
				"pve-services:22",
				"pve-services:8006",
				"heimdall:22",
				"services-host:22",
			],
		},
		{
			"src":  "services-host",
			"deny": ["pve-env1:22", "pve-env1:8006", "jetson:22", "heimdall:22"],
		},
		{
			"src":  "pve-env1",
			"deny": ["pve-env2:22", "pve-services:22", "pve-services:8006"],
		},
		{
			// dev-git can't reach anything else on the tailnet
			"src": "dev-git",
			"deny": [
				"pve-env1:22",
				"pve-env1:8006",
				"pve-services:22",
				"pve-services:8006",
				"services-host:22",
				"heimdall:22",
				"jetson:22",
				"workstation:22",
			],
		},
		{
			// pve-ansible reaches Gitea SSH on dev-git and nothing else there
			"src":    "pve-ansible",
			"accept": ["dev-git:2222"],
			"deny":   ["dev-git:22", "dev-git:3000"],
		},
	],
}
```

## Apply
1. Be on the LAN, not relying on the tailnet, while you do this.
2. Open `https://login.tailscale.com/admin/acls/file`.
3. Edit the live policy in place rather than pasting a whole block over it, unless this copy is confirmed current. If a test fails, Tailscale rejects the save and names the failing test.
4. From the workstation over Tailscale, confirm SSH to each destination and the Proxmox web UI on 8006 for the four PVE nodes.
5. Confirm Tailscale SSH still works to [[heimdall]].

## Rollback
Policy changes are logged in the configuration audit logs and can be reverted from there. Tailscale's own reset option restores the original allow-all default.

## Not covered here
The bastion rules live outside Tailscale (OPNsense, no Tailscale client on the bastion, no route advertisement on [[pve-services]]). See [[tailscale-access-model]].

## Related
- [[Tailscale]]
- [[tailscale-access-model]]
- [[homelab-update-automation]]

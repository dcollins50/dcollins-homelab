---
name: session-2026-09-25-utc-rollout
type: session
date: 2026-09-25
---

Execution of [[standardize-node-clocks-utc]], following the SSH key rollout in [[session-2026-09-25-ssh-key-rollout]]. Goal: actually run the `timedatectl set-timezone UTC` change across the fleet.

## Script issues hit and fixed, in order

1. `BatchMode=yes` rejected every host WSL hadn't previously connected to (all 4 Proxmox hosts, Heimdall, both LXC-parent SSH calls), failing with "Host key verification failed" instead of prompting. These hosts had only ever been reached from Windows/PowerShell before, which has its own separate `known_hosts` from WSL. Fixed with `-o StrictHostKeyChecking=accept-new`.
2. `sudo` over a plain `ssh host "command"` has no TTY to prompt on, failed with "a terminal is required to read the password." Fixed with `ssh -t`.
3. With `-t` fixed, the sudo prompt just read `[sudo] password for admin:` with no hostname, ambiguous across the 4 VMs that all use the `admin` account (pve-iris, soc-stack, pve-ca-intermediate, soar-host). Fixed using `sudo -p` to build a custom prompt string with the hostname baked in: `[sudo] password for admin@$name: `.
4. Heimdall's username was a guess (`pi`) since it was never recorded in the vault. Failed with "Permission denied (publickey)." Corrected to `admin`, confirmed working.

## Result: 16 of 16 attempted hosts confirmed UTC

**4 Proxmox hosts:** pve-gateway, pve-services, pve-env1, pve-env2
**Heimdall:** confirmed, username corrected to `admin` and now documented on its host note (previously had no login recorded at all)
**5 VMs:** pve-iris, soc-stack, services-host, pve-ca-intermediate, soar-host
**6 LXCs** (via `pct exec` from their parent Proxmox host, no direct SSH): authentik (2201), pve-int-stepca (511), pve-bastion (2202), pve-authtunnel (2100) — all four on pve-services — and npm-dmz (2200) on pve-env1

wazuh-manager was already UTC prior to this work (see [[session-2026-09-24-alert-engineering-baseline]]), not re-touched.

Each host's result was read directly off `timedatectl | grep 'Time zone'` output in the same command that set it, not assumed from exit status.

## Remaining (not touched in this session)

- **kali-attack, ubuntu-401, pve-ca-root** — were stopped during the Sep 25 SSH key rollout, so never got a key, and were excluded from this run for the same reason. Same script covers them once powered on and key-rolled.
- **OPNSense** — FreeBSD, not `timedatectl`. Web UI change (System > Settings > General) still pending.
- **metasploitable2, DVWA** — intentionally excluded, staying on password auth by design (see [[session-2026-09-25-ssh-key-rollout]]).
- **malware-win11** — Windows, air-gapped, needs `tzutil` via console access, not SSH.
- **docker-host-template** — stopped VM template, would need to be booted temporarily or edited offline.

## Related
- [[standardize-node-clocks-utc]]
- [[session-2026-09-25-ssh-key-rollout]]

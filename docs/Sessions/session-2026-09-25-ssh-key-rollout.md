---
name: session-2026-09-25-ssh-key-rollout
type: session
date: 2026-09-25
---

Follow-up from the [[standardize-node-clocks-utc]] decision. Goal: get SSH key auth in place on the VMs that were still password-only, as a prerequisite for scripting the UTC timezone change (and future health-check automation generally).

## Auth inventory taken

Checked login method across all 17 hosts Daniel gave credentials for. Split:
- 4 Proxmox hosts + Heimdall: already SSH key auth
- 5 LXCs (pve-bastion, pve-int-stepca, npm-dmz, pve-authtunnel, authentik): accessed via `pct enter`/`pct exec` from the Proxmox host, no SSH key needed
- 9 VMs: password auth, candidates for key rollout
- metasploitable2, DVWA: intentionally left on password auth (lab targets, never copy a key here)
- malware-win11: air-gapped, not applicable

## WSL setup blockers hit and resolved

- WSL has its own home directory, separate from Windows — `~/.ssh` didn't exist in WSL even though a key existed under `/mnt/c/Users/Owner/.ssh/`
- Copied `id_ed25519`/`id_ed25519.pub` into WSL's `~/.ssh`, fixed permissions (`chmod 700` on the dir, `600` on the private key — files under `/mnt/c/` mount world-writable by default, which SSH refuses)
- `ssh-agent` wasn't running in the WSL session; `ssh-add -l` failed with "could not open a connection to your authentication agent" until `eval "$(ssh-agent -s)"` was run
- The private key itself is passphrase-protected; `ssh-add` prompted for it, resolved by entering it
- Confirmed: `ssh-agent` state does not persist across WSL terminal restarts. This whole sequence (start agent, add key, enter passphrase) has to be repeated in any fresh WSL session before key auth works from that terminal.

## Rollout run

9 candidate VMs, 3 were stopped at check time (kali-attack, ubuntu-401, pve-ca-root, per a Proxmox summary screenshot Daniel shared) and excluded from the run. 6 ran:

- pve-iris, soc-stack, services-host, soar-host, wazuh-manager: key installed clean
- pve-ca-intermediate: first attempt looked interrupted (Ctrl-C during host-key prompt) but had actually already written the key; second run correctly reported "already exists on remote"

**Verified, not just trusted `ssh-copy-id`'s own report:** ran all 6 with `ssh -o BatchMode=yes -o ConnectTimeout=5 <host> "hostname"`, which fails outright rather than falling back to a password prompt. All 6 returned their hostname cleanly. Genuine passwordless key auth confirmed on pve-iris, soc-stack, services-host, pve-ca-intermediate, soar-host, wazuh-manager.

## Remaining

- kali-attack, ubuntu-401, pve-ca-root: still password auth, rollout not run (were stopped). Same script covers them once powered on.
- npm-dmz, pve-authtunnel, authentik usernames still unconfirmed (Daniel accesses these via `pct enter`, hasn't needed to know the login).
- The UTC timezone change itself (the original goal that led here) has not been done yet — this session only cleared the auth blocker.

## Related
- [[standardize-node-clocks-utc]]
- [[session-2026-09-25-rule-100004-confirmation]]

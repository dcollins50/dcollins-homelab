---
name: sop-harden-debian-vps
type: sop
status: complete
---

# SOP: Harden a Fresh Debian VPS

**Status:** Complete procedure (first run Sep 28, 2026 on [[vps-redirector]])
**Objective:** Take a bare, provider-default Debian VPS (root SSH, password auth, no firewall) to a baseline hardened state: a non-root sudo user, key-only SSH, root login and passwords off, a default-deny firewall, automatic security patches, and brute-force protection.

---

## Why this is an SOP, not a single runbook

This is dependency-ordered work across several tools that must happen in the right sequence or you lock yourself out. It touches user/account setup, OpenSSH config (including provider-shipped drop-ins that override the main file), `nftables`, `unattended-upgrades`, and `fail2ban`. Each step has a lockout-safety checkpoint that must pass before the next step, because a mistake in SSH or firewall config on a remote box with no console is not recoverable over SSH.

## The one rule that prevents lockout

**Keep your original working session open until the very end.** Every time you change SSH or firewall config, prove the new state works in a *second, separate* login before you close the first one. If the new session fails, you still have the old one to undo the change. Never log out of your only working session right after a change.

## Steps

### 1. Create a non-root sudo user

```bash
adduser admin
usermod -aG sudo admin
```

### 2. Set up key-only login for that user

Generate a dedicated key for this box on your workstation (do not reuse a key that already protects other machines):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vps_key -C "vps-redirector"
```

Install the public key for the new user. Either `ssh-copy-id -i ~/.ssh/vps_key.pub admin@<vps-ip>`, or paste the `.pub` contents into `/home/admin/.ssh/authorized_keys` (mode `600`, owned by `admin`, `.ssh` mode `700`).

**Checkpoint:** In a new terminal, `ssh -i ~/.ssh/vps_key admin@<vps-ip>` and confirm you land in a shell and `sudo -v` works. Do not proceed until this succeeds. This is the session you will keep open for the rest of the SOP.

### 3. Disable root login and password auth

Set the policy in a single hardening drop-in rather than editing the main config by hand. OpenSSH reads `/etc/ssh/sshd_config.d/*.conf`, files are read in lexical order, and the FIRST value seen for a directive wins. So name the hardening file so it sorts before any provider file: `00-hardening.conf`.

Write `/etc/ssh/sshd_config.d/00-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers admin
```

**Provider drop-in gotcha (this is what bit the provider):** the provider ships `/etc/ssh/sshd_config.d/99-ctrl.conf` forcing `PermitRootLogin yes` and `PasswordAuthentication yes`. Because `00-hardening.conf` sorts first, it already wins on a fresh box, but remove the conflicting file anyway so the box has one source of truth. Also comment any uncommented `yes` lines in the main `sshd_config`. Check what is in play first:

```bash
sudo grep -riE 'PermitRootLogin|PasswordAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
sudo rm -f /etc/ssh/sshd_config.d/99-ctrl.conf
sudo sed -i -E 's/^\s*(PermitRootLogin\s+yes)/#\1/I; s/^\s*(PasswordAuthentication\s+yes)/#\1/I' /etc/ssh/sshd_config
```

Confirm the intended values are the only ones that survive, then validate and reload:

```bash
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'
```

The `sshd -T` output is the real proof, not login behavior. It must read `permitrootlogin no`, `passwordauthentication no`, `allowusers admin`.

**Checkpoint:** From a new terminal, confirm `ssh root@<vps-ip>` is refused and password login is refused (`ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@<vps-ip>` should fail). Your key session from step 2 must still be open and working. Only once both refusals are confirmed do you continue.

### 4. Default-deny firewall with nftables

Allow only established traffic, loopback, and inbound SSH. Everything else inbound is dropped.

```bash
sudo apt update && sudo apt install -y nftables
sudo systemctl enable --now nftables
```

Write `/etc/nftables.conf`:

```
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    iif "lo" accept
    ct state established,related accept
    tcp dport 22 accept
    ct state invalid drop
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output { type filter hook output priority 0; policy accept; }
}
```

**Checkpoint before applying:** confirm the SSH `accept` line is present and correct, because `policy drop` with no SSH allow will cut you off the moment it loads. Then apply and persist:

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl restart nftables
```

**Checkpoint after applying:** open one more new SSH session and confirm it connects. Keep the prior session open while you test.

### 5. Automatic security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Confirm the security origin is enabled in `/etc/apt/apt.conf.d/50unattended-upgrades` and that `/etc/apt/apt.conf.d/20auto-upgrades` has `Update-Package-Lists` and `Unattended-Upgrade` set to `"1"`.

### 6. Brute-force protection with fail2ban

```bash
sudo apt install -y fail2ban
```

Create `/etc/fail2ban/jail.local` with an `[sshd]` jail enabled (a `bantime`, `findtime`, and `maxretry` of your choosing; the defaults are a reasonable start). Then:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

The status output confirms the jail is watching the SSH log.

## Final verification

- `ssh root@<vps-ip>` refused, password auth refused, key login works.
- `sudo nft list ruleset` shows `policy drop` on input with an SSH allow.
- `systemctl is-active nftables fail2ban unattended-upgrades` all report active/enabled.
- Only now, close every extra session. The box is at baseline.

## Automating this: the bootstrap script

Every step above is bundled into an idempotent script at `Scripts/bash/harden-debian-vps.sh` in the repo (gitignored, local only). Set the three variables at the top (`ADMIN_USER`, `SSH_PORT`, `SSH_PUBKEY`) and run it as root:

```bash
sudo bash harden-debian-vps.sh
```

It is safe to re-run: anything already correct is left alone. The lockout-proof way to run it is from the provider web console (the provider noVNC/serial) as root, so there is no SSH session to lose when password auth is turned off. The script refuses to lock down SSH unless the admin key is actually in place. The manual steps here remain the reference for what the script does and for doing it by hand when you want to understand a step.

## What this SOP does not cover

This is host-level baseline hardening only. It does not cover the transport tunnel back to the homelab, the OPNSense-side firewall scoping, or any service the box runs. For the the provider redirector specifically, those live with [[vps-redirector]] and, once built, the WireGuard tunnel note.

## Related
- [[vps-redirector]]
- [[Security-Lab]]

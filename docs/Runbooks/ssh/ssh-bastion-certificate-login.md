---
name: ssh-bastion-certificate-login
type: runbook
tool: ssh
---

# Runbook: SSH into the bastion with a step-ca certificate (Windows workstation)

Tested on Oct 3, 2026 from the Windows workstation, twice (the second time after the first certificate's overwrite prompts were answered). Background and rules: [[bastion-ssh-certificate-login]]. Host: [[pve-bastion]].

## Before you start
- The Cloudflare One client must be connected on the workstation. The bastion is only reachable over the Mesh at `100.x.x.x` ([[cloudflare-warp-windows-workstation-mesh-access]]).
- `step` CLI installed and bootstrapped to `https://stepca.homelab.local:9000` (done Oct 3, 2026).
- A certificate lasts about 16 hours (the provisioner default, not pinned). After that, get a new one.

## Steps
1. **Workstation, PowerShell.** Get a certificate. A browser tab opens for the Authentik login.
   ```
   step ssh certificate user@example.com "$env:USERPROFILE\.step\ssh-test-key" --provisioner Authentik --no-agent --no-password --insecure
   ```
   If the files already exist, step asks three overwrite questions (private key, public key, certificate). Answer `y` to each. The identity label in the command does not matter: step-ca takes the identity and principals from the Authentik login.
2. **Workstation, PowerShell.** Log in:
   ```
   ssh -i "$env:USERPROFILE\.step\ssh-test-key" -o CertificateFile="$env:USERPROFILE\.step\ssh-test-key-cert.pub" -o IdentitiesOnly=yes [admin-user]@100.x.x.x
   ```
3. To check the certificate's contents: `ssh-keygen -L -f "$env:USERPROFILE\.step\ssh-test-key-cert.pub"`.

## Not tested
- `step ssh login` with the Windows ssh-agent (the intended everyday flow).
- An SSH config alias so the login is one short command.
- The private key has no passphrase (`--no-password --insecure`) and the file is still named `ssh-test-key`. A stolen key alone is useless without a valid certificate, but a stolen key plus an unexpired certificate works until it expires.

## If it fails
| Symptom | Likely cause | What to do |
| --- | --- | --- |
| `Permission denied (publickey)` | Certificate expired, or the principal does not match `/etc/ssh/auth_principals/[admin-user]` | Run step 1 again, then check the principals with step 3 |
| Timeout or connection reset | Cloudflare One client not connected, or the Gateway Allow policy is off | Connect the client, then see [[bastion-host-firewall]] |
| Browser shows Authentik redirect or `invalid_client` errors | Authentik provider `step-ca` redirect URI or the provisioner client secret changed | See [[session-2026-10-03-stepca-ssh-ca]] |
| Nothing works | Certificates or sshd broken | `sudo pct enter 2202` from [[pve-services]]. Passwords come back by deleting `/etc/ssh/sshd_config.d/20-no-passwords.conf` and running `systemctl restart ssh` (never reload) |

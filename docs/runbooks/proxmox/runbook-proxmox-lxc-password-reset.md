# Runbook: Reset a Password or Add a User on an LXC (No Existing Credentials)

**Category:** Proxmox
**When to use:** Locked out of an LXC (forgotten password, or the container never had a non-root user set up), and you have root/admin access to the Proxmox host itself.

## Why this works

The Proxmox host has root-level control over every LXC's filesystem and process namespace running on it. You don't need the container's own credentials — you reach in from the host directly.

## Steps

**1. Identify the correct VMID.** Confirm you have the right container before touching it — `pct list` on the host, or check the Proxmox UI. Don't guess from memory if there's any ambiguity between similarly-named containers.

**2. Enter the container from the host, bypassing login entirely:**
```bash
pct enter <vmid>
```
This drops you into a root shell inside the container. No password prompt, no existing credentials needed.

**3a. To reset an existing user's password:**
```bash
passwd <username>
```

**3b. To add a new user that doesn't exist yet:**
```bash
adduser <username>
```
(interactive — walks you through setting the password and creating the home directory)

or non-interactively:
```bash
useradd -m -s /bin/bash <username>
passwd <username>
```

**4. Grant sudo if the user needs it:**
```bash
usermod -aG sudo <username>
```

**5. Exit back to the host:**
```bash
exit
```

## Alternative: without an interactive shell

If you want to reset a password without entering the container interactively (e.g. scripting this), use `pct exec` instead of `pct enter`:
```bash
pct exec <vmid> -- passwd <username>
```

## Notes

- You cannot recover the *old* plaintext password this way — it's stored as a hash in `/etc/shadow` and isn't reversible. This resets it to a new value; it doesn't reveal the old one.
- This works regardless of whether the container is privileged or unprivileged — `pct enter`/`pct exec` operate at the Proxmox host level, above the container's own permission model.

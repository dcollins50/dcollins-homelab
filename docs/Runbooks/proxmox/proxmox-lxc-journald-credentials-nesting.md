---
name: proxmox-lxc-journald-credentials-nesting
type: runbook
tool: proxmox
---

# Runbook: journald dead in a Debian 13 unprivileged LXC (243/CREDENTIALS)

**When to use:** `journalctl` in a Debian 13 container says "No journal files were found", or services log nothing. Hit on [[pve-authtunnel]] on Oct 2, 2026, which hid the `cloudflared` logs during a 502 investigation. The same bug was fixed on [[pve-bastion]] on Sep 21.

## Diagnose
Inside the container:
```bash
systemctl status systemd-journald
```
Look for `status=243/CREDENTIALS`.

## Fix
On the Proxmox node that hosts the container (pve-services for 2100 and 2202), enable nesting and reboot the container:
```bash
sudo pct set <vmid> --features nesting=1
sudo pct reboot <vmid>
```
Then confirm `journalctl -u <service>` returns entries.

## Prevent
Set `nesting=1` at build time for every Debian 13 unprivileged LXC. Add it to the checklist in [[proxmox-lxc-build-pattern]].

## Related Documentation
- [[proxmox-lxc-build-pattern]]
- [[proxmox-create-lxc]]

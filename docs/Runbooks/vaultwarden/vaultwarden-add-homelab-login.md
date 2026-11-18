---
name: vaultwarden-add-homelab-login
type: runbook
tool: vaultwarden
---

House style for adding or renaming a homelab item in Vaultwarden. These are the decisions Daniel made on Oct 3, 2026, see [[vaultwarden-integration]].

## Where it goes
All homelab items live under the `homelab/` folder, in one of seven subfolders. Vaultwarden nests with a slash in the folder name, and the plain `homelab` folder stays as the parent.
- `homelab/network`: firewall, switch, DNS, VPN gear.
- `homelab/identity`: identity and certificate services, for example Authentik and step-ca.
- `homelab/soc`: SIEM, SOAR and IR tools.
- `homelab/apps`: other web apps and services.
- `homelab/hosts`: SSH and console logins for machines, including Proxmox.
- `homelab/keys`: SSH key passphrases, see [[vaultwarden-store-ssh-key-passphrases]].
- `homelab/cloud`: cloud accounts and VPS.

## How it is named
- Web UI logins: named after the service, taken from its web address. Examples: `iris`, `shuffle`, `portainer`, `soc-stack`. Filed by function.
- SSH and console logins: named after the host, for example `pve-iris` or `soar-host`. Filed in `homelab/hosts`.
- Do not repeat the username in the name, since the vault list shows it under the name.
- Two logins on one name, such as an admin and a root login, differ by username.
- The web UI of a service and the host it runs on are different items. `pve-iris` is an SSH login, the IRIS web UI is `iris`.

## Fields
- Username and password. Generate the password in the vault.
- URI: the exact address you use. Internal items on the tailnet use their tailnet address.
- Notes: what the login is for and where it is recorded. Never paste another secret into notes.

## Steps
1. New item, type Login.
2. Fill the fields as above, pick the folder.
3. Save.
4. If you rotated a password on a system, update the item the same day.

## Related
- [[vaultwarden]]
- [[vaultwarden-integration]]

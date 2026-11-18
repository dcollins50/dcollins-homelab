---
name: vaultwarden-use-cli-in-scripts
type: runbook
tool: vaultwarden
---

Pull secrets out of Vaultwarden inside scripts and automation without storing passwords in plain files. Set up the CLI first: [[vaultwarden-bitwarden-cli-setup]].

## Two ways to log in
- Interactive, `bw login`: for a person at a keyboard.
- API key, `bw login --apikey`: for automation. It uses a client id and secret. Find yours in the web vault, Account settings, Security, View API key (the CLI docs describe this flow, confirm the menu names on your version). Export them as environment variables so the script does not prompt:
  ```bash
  export BW_CLIENTID="<client_id>"
  export BW_CLIENTSECRET="<client_secret>"
  ```
  Even after an API key login, the vault is locked. It still needs `bw unlock`, which needs the master password. That is by design, and it means unattended automation has to keep the master password somewhere. Decide where: <FILL IN>. Keep it out of the repo and out of shell history.

## Pattern for a script
```bash
#!/usr/bin/env bash
set -euo pipefail
bw login --apikey --nointeraction > /dev/null 2>&1 || true
export BW_SESSION="$(bw unlock --passwordenv BW_PASSWORD --raw)"
bw sync > /dev/null
SECRET="$(bw get password '<item name>')"
# use "$SECRET" here
bw lock > /dev/null
```
Check the flags `--nointeraction`, `--passwordenv` and `--raw` against `bw unlock --help` on the installed version, since they were not confirmed from the docs page during this write-up.

## Getting things out
- A password: `bw get password <item name or id>`.
- A username: `bw get username <item name or id>`.
- A full item as JSON: `bw get item <id>`.
- A list: `bw list items --folderid <id>`, or `bw list items --search <text>`.
- Naming helps here. Items are named by service or host, see [[vaultwarden-add-homelab-login]].

## Cautions
- Do not echo secrets into logs. Shell tracing (`set -x`) prints them.
- Do not put the master password in the script file.
- Prefer a dedicated automation account with access to only what it needs, once organizations and collections are in use. Not chosen yet, so today an automation account would see its whole personal vault.
- For Ansible later, check whether a lookup plugin is a better fit than a wrapper script. Not researched yet.

## Related
- [[vaultwarden-bitwarden-cli-setup]]
- [[vaultwarden]]

Source: https://bitwarden.com/help/cli/

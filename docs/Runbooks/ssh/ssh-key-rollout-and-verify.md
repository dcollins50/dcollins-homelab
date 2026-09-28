---
name: ssh-key-rollout-and-verify
type: runbook
tool: ssh
---

Roll SSH key auth out to a list of hosts still on password auth, and actually confirm it worked, rather than trusting `ssh-copy-id`'s own success message. Covers three gotchas hit rolling this out to 6 VMs on Sep 25, 2026, each of which silently breaks a naive script.

## Prerequisite

A loaded SSH agent with the right key, see [[ssh-wsl-agent-setup]] if running from WSL.

## Gotcha 1: unknown host keys break non-interactive scripts

A host WSL (or any client) has never connected to before prompts interactively to accept its host key. In a scripted loop, this either hangs waiting for input or, worse, silently fails if stdin isn't a real terminal, showing as "Host key verification failed" with no prompt ever shown.

Fix: `-o StrictHostKeyChecking=accept-new` auto-trusts a host's key the first time it's seen, without prompting. This is a reasonable trust model for hosts already being administered directly (you're not blindly trusting a random host, you're connecting to infrastructure you already control), but it does mean no manual fingerprint verification happens, know that tradeoff before using it broadly.

## Gotcha 2: `sudo` over a non-interactive SSH command has no TTY to prompt on

`ssh host "sudo some-command"` fails with `sudo: a terminal is required to read the password` because there's no pseudo-terminal allocated for the password prompt.

Fix: add `-t` to force TTY allocation:
```bash
ssh -t host "sudo some-command"
```

## Gotcha 3: an ambiguous sudo prompt across multiple same-named accounts

With `-t` fixed, the sudo prompt just reads `[sudo] password for <user>:` with no hostname. If several hosts in a loop share the same account name (e.g. several VMs all logging in as `admin`), every prompt looks identical, easy to lose track of which host's password is actually being typed.

Fix: use `sudo -p` to build a custom prompt string with the hostname baked in:
```bash
ssh -t host "sudo -p '[sudo] password for admin@\$name: ' some-command"
```

## Full rollout script pattern

Putting the pieces together, for a batch of hosts:
```bash
declare -A hosts=(
  ["hostname1"]="user@ip1"
  ["hostname2"]="user@ip2"
)

for name in "${!hosts[@]}"; do
  target="${hosts[$name]}"
  echo "=== $name ($target) ==="
  ssh-copy-id "$target"
  echo
done
```
`ssh-copy-id` on its own is usually enough for the key-copy step; the `-t`/`accept-new`/`sudo -p` gotchas above matter for any *follow-up* command run over SSH after the key's in place (like the actual config change the key rollout was for), not for `ssh-copy-id` itself.

## Verifying key auth actually works, don't just trust the tool's report

`ssh-copy-id` can report "all keys skipped, already exist on remote" when a prior run was interrupted mid-way (e.g. Ctrl-C during a host-key prompt) but had actually already written the key before being killed. Don't take that message as proof key auth is broken, and don't take a clean "Number of key(s) added: 1" as proof it's *working* either, both are the tool's own bookkeeping, not a live test.

Actually verify with a command that fails outright instead of falling back to a password prompt:
```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 user@host "hostname"
```
A clean hostname printed back confirms genuine passwordless key auth. `BatchMode=yes` means if key auth doesn't work, SSH fails immediately instead of silently prompting for a password (which would make a script look "successful" even when it's actually falling back to a password every time, not really key auth).

## Related
- [[ssh-wsl-agent-setup]]
- [[session-2026-09-25-ssh-key-rollout]]

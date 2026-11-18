---
name: ansible-verify-signed-commit
type: runbook
tool: ansible
---

Prove that the commit about to be run on a server was signed by the author, before any playbook runs. Do this on [[pve-ansible]] every time, after pulling.

## Why
The control node holds the key that reaches the server. A playbook is code that runs as root there. If the repo were tampered with, the first sign would be a commit that does not verify.

## Steps (on pve-ansible, inside the repo)
```bash
git pull
git verify-commit HEAD
```
Expected: git reports a good signature for the author's signing key. If it says the signature is bad, unknown or missing, stop. Do not run the playbook. Find out why first.

## If it fails
- Check git is pointed at the trust file: `git config --get gpg.ssh.allowedSignersFile` should print `/etc/ansible-trust/allowed_signers`.
- The file is root owned on purpose, so a normal account cannot add its own key to it.
- On the workstation, git shows "No signature" because the workstation has no trust file. That is expected. The real check is this one, on the control node.

Next: [[ansible-load-ssh-agent-key]].

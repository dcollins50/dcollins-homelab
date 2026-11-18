---
name: ansible-switch-os-account
type: runbook
tool: ansible
---

Turn an operating system login on the SOC server off, back on, or delete it. Each person is an entry in `baseline_admin_users` in the inventory's `group_vars/all.yml`. Run the playbook from [[pve-ansible]] after [[ansible-load-ssh-agent-key]], and verify the commit first with [[ansible-verify-signed-commit]].

## The two switches
Both are optional on a person's entry and default to a normal, enabled account.
- `enabled: false` locks the account. The shell becomes nologin, the groups, SSH keys and sudo rule are removed, the account expires, the person is dropped from the SSH allow list and any open session is ended. The account and its home folder stay.
- `enabled: false` with `remove: true` deletes the account and its home folder.
- Removing the line `enabled: false` turns the account back on. The keys come from the inventory, so nothing else is needed.

The Ansible service account has no switch. The play refuses a list where it is disabled, removed or missing, because that would cut Ansible off and only the provider console could fix it. `remove` without `enabled: false` is also refused.

## Steps
1. Edit the person's entry in `group_vars/all.yml`, for example add `enabled: false`.
2. Commit with a signature, push, then `git pull` and verify the commit on the control node.
3. Dry run, then the real run, as the service account (root login is closed):
```bash
ansible-playbook playbooks/01-baseline.yml -u ansible --check --diff
ansible-playbook playbooks/01-baseline.yml -u ansible
```
4. Run it once more. A second run must report `changed=0`.

A dry run changes nothing, including sessions. A disabled account shows in the dry run as the lock, key and sudo tasks changing, and the SSH allow list losing the name.

## Check on the server
```bash
sudo bash -c 'getent passwd NAME | cut -d: -f1,7; passwd -S NAME; chage -l NAME | grep "Account expires"; ls /etc/sudoers.d; ls -A /home/NAME/.ssh; grep AllowUsers /etc/ssh/sshd_config.d/00-achilles-hardening.conf'
```
A locked account shows the nologin shell, status `L`, an expiry in 1970, no sudo file, no keys and no name in `AllowUsers`.

## Tested
On 2026-10-25 on the live server: dry run, lock, rerun with no change, unlock, rerun with no change. The delete path (`remove: true`) was tested on throwaway accounts on a scratch machine, not on the live server.

Next: [[ansible-run-playbook]].

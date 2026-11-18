---
name: ansible-load-ssh-agent-key
type: runbook
tool: ansible
---

Load the passphrase protected `ansible_soc` key into an agent so playbooks can reach the server. Do this on [[pve-ansible]] at the start of every session.

## Steps (on pve-ansible)
In a brand new shell there is no agent yet:
```bash
eval "$(ssh-agent -s)"
```
Then load the key for one hour:
```bash
ssh-add -t 3600 ~/.ssh/ansible_soc
ssh-add -l
```
`ssh-add -l` should list the key. Type the passphrase when asked.

## When you are done
Stop the agent this shell started, so it does not pile up with the ones from earlier sessions:
```bash
ssh-agent -k
```
It stops the process named in `$SSH_AGENT_PID` and forgets the loaded key. To see what is left running, use `pgrep -a ssh-agent`. Any agent that is not the one in `$SSH_AGENT_PID` is a leftover and can be stopped with `kill <pid>`.

## Notes
- The hour limit is deliberate. A run that starts after expiry fails with "Permission denied (publickey)". Load the key again.
- Each `eval "$(ssh-agent -s)"` starts another agent process. Reuse the shell where you can, and see the section above for stopping old ones.
- The key path is set in the repo's `ansible.cfg`, so playbooks need no `-i` key option.

Next: [[ansible-run-playbook]].

---
name: semaphore-for-update-automation
type: decision
status: live
---

## Decision (Nov 7, 2026)
The new homelab Ansible container runs its playbooks through Semaphore. Terraform is not used for this. See [[homelab-update-automation]].

## Why
The job is rolling out updates to the Proxmox nodes and their guests, plus running the Asscord deploy playbooks. That is Ansible work: log in to machines and change what is on them. Semaphore is a web UI and scheduler for Ansible playbooks, with manual runs, schedules, task logs and an encrypted key store.

Terraform creates and destroys infrastructure (for example making an LXC through the Proxmox API). It does not log in to machines to update them, so it does nothing for this job.

## Ruled out
- Terraform as the way to run updates: wrong tool for the job, see above.
- CLI only Ansible on this box (the Oct 15, 2026 plan for a homelab control node): replaced by Semaphore here because scheduled runs are wanted.

## Left open
Semaphore can also run Terraform or OpenTofu. If LXCs are ever defined as code, that can run from the same UI. Semaphore does not store Terraform state itself; that would need its own home.

## Not affected
[[pve-ansible]] stays CLI only for the Achilles work, per [[achilles-soc-ansible]].

---
name: remote-access-path-order
type: decision
status: live
---

## Decision
Remote admin access has three paths, in this order (set by Daniel, Sep 24, 2026):
1. **Primary:** Cloudflare WARP into the SSH bastion ([[pve-bastion]]), with Authentik and a step-ca issued SSH cert as the two MFA layers. The bastion is built, but as left in the Sep 21 session WARP enrollment is blocked by a QUIC error, sshd cert trust is not configured and the SSH cutover has not happened, so this path is not usable yet. Of the three paths, only Tailscale is recorded as working today.
2. **Secondary:** WireGuard on [[heimdall]]. In progress. The endpoint config is stale and not yet properly configured.
3. **Tertiary:** Tailscale, the break-glass path. See [[tailscale-access-model]] for the rules on it.

## Replaces
Earlier notes gave conflicting orders. [[Network]] called WireGuard primary and live. [[Tailscale]] called WireGuard tertiary. [[heimdall]] called all three break-glass. All three were updated to match this note.

## Related
- [[Network]]
- [[Tailscale]]
- [[tailscale-access-model]]
- [[sop-ssh-bastion-build|SSH Bastion Build]]

---
name: session-2026-10-02-bastion-warp-enrollment
type: session
date: 2026-10-02
---

Picked up the stalled [[pve-bastion]] WARP enrollment from [[session-2026-09-21-stepca-and-bastion]]. Build plan is [[sop-ssh-bastion-build]].

## What was done
- Found the root cause of the enrollment failure: Nginx Proxy Manager returned a 502 on the Authentik authorize redirect, `upstream sent too big header`. Fixed with larger proxy buffers on the `authentik` host. See [[npm-fix-502-upstream-header-too-big]]. The Authentik login in the WARP flow now completes.
- No logs were visible on [[pve-authtunnel]] because journald was dead (243/CREDENTIALS). Fixed with `nesting=1` and a container reboot. See [[proxmox-lxc-journald-credentials-nesting]].
- Enrolled the bastion by the headless token hand-off, then converted it from a user device to a Mesh node (`bastion-node`, 100.x.x.x). See [[cloudflare-warp-mesh-node-headless]].
- Decision: Mesh IP only for now, no advertised routes and no IP forwarding. SSH over the Mesh IP bypasses OPNsense inbound rules, so the controls become Cloudflare policies, sshd, and a host firewall on the bastion.

## What went wrong (for next time)
- I guessed OPNsense was blocking UDP 443. The tunnel was up, so that was wrong.
- I told Daniel clicking "Open in Cloudflare One Client" would enroll the host. A headless box needs the token hand-off.
- The enrollment token expires in about 60 seconds and one was pasted into chat. Paste tokens only on the host.
- A registration was invalidated (`ApiMismatch`) after a rushed second attempt. One clean attempt beats several.
- Several times a stale command output was pasted instead of the new one. Send commands as plain text, one at a time.

## Left open at the end of the session
- The node does not stay connected: the WARP DNS proxy cannot bind an IPv6 address because IPv6 is disabled in the container. `warp-cli disconnect` was run to stop the loop, so the bastion is currently disconnected. Choose a fix from [[cloudflare-warp-mesh-node-headless]].
- The last `warp-svc` log showed `RegistrationMissing(Invalidated(ApiMismatch))` on the host and NTP timeouts in the WARP client. Cause not investigated. Check whether the node needs re-enrolling.
- Still not done: enroll the workstation WARP client, test `ping 100.x.x.x` and `ssh [admin-user]@100.x.x.x`, check Mesh access policies, sshd `TrustedUserCAKeys` for step-ca, remove the temporary workstation to bastion port 22 rule.

---
name: session-2026-10-29-stepca-key-check-acme-decision
type: session
date: 2026-10-29
---

Raw log. Prompted by a Reddit comment about DNS-01 wildcard certs on a public subdomain, which looked simpler than the current PKI.

- Confirmed VM 501 ([[pve-ca-intermediate]]) still issues certs for every service deployed before step-ca existed.
- Checked whether [[pve-int-stepca]] holds the same intermediate key by comparing `openssl x509 -noout -pubkey | sha256sum` on both. Result: identical (`89a5901a3d8dae78...`). Same key pair on two online machines.
- Decision: ACME for leaf issuance. See [[stepca-acme-leaf-issuance]].

## What went wrong along the way
- Pasted a command with a literal `<path from above>` placeholder. Bash treated it as a redirect and openssl read nothing, so both hosts printed `e3b0c442...` (the SHA256 of empty input). That value means "no input", not a match.
- `sudo` is not installed on pve-int-stepca. A `find` with `2>/dev/null` hid that error and looked like "no files found". Run commands for that container from pve-services with `sudo pct exec 511 -- ...`.

## Next
- Pick the first service to move and the ACME challenge type. Check firewall paths before enabling anything.

## Later in the session
- DNS state found: pve-ca-intermediate resolves through Pi-hole (192.168.100.x, then 1.1.1.1), pve-int-stepca through Unbound (10.0.30.x). `stepca.homelab.local` resolves to 10.0.30.x from 511 only. `stepca.homelab.internal` resolves nowhere. See [[pve-ca-intermediate]], [[pve-int-stepca]].
- Uptime Kuma is not monitoring cert expiry, contrary to the earlier [[services-host]] note. Corrected.
- Live cert check results are in [[pve-ca-intermediate]]. Earliest known expiry: Feb 8, 2027 (pve-gateway).
- Daniel asked whether DNS rollout should come before certs. Open, leaning DNS phases 0 and 1 first.

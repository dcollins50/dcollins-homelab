# Runbook: Diagnose Network Connectivity on a Minimal Host (No curl Yet)

**Category:** Proxmox / Linux
**When to use:** Troubleshooting DNS or TCP reachability on a fresh minimal container/host where `curl` isn't installed yet — including the common chicken-and-egg case where the problem you're diagnosing is *why `apt install curl` itself is failing*.

## DNS resolution check

```bash
getent hosts <hostname>
```
Works on essentially any Linux base with no extra tooling. Returns the resolved IP(s), or nothing/an error if resolution fails. Note: this shows what DNS *returns*, not whether the app will actually try to connect to it — a host can have a working AAAA (IPv6) record that resolves fine here while the actual connection attempt fails because there's no IPv6 route (see the LXC build pattern runbook for that specific failure mode).

## TCP connectivity check (no curl, no netcat)

Bash's own `/dev/tcp` pseudo-device can open a raw TCP connection with no extra tools at all:
```bash
timeout 5 bash -c "cat < /dev/null > /dev/tcp/<host>/<port>" && echo "CONNECTED" || echo "FAILED"
```
This confirms whether a TCP handshake succeeds on a given port — useful for distinguishing "DNS resolves fine but the firewall/routing is blocking the actual connection" from "DNS itself is the problem," and for confirming a specific firewall rule is actually working before installing anything that depends on it.

## Narrowing further once you have signal

- **DNS fails, TCP check not yet relevant:** check `/etc/resolv.conf` for the configured nameserver, and confirm that specific resolver is reachable from this host's VLAN (a resolver on an unreachable network looks identical to "DNS is broken" from the application's point of view).
- **DNS succeeds, TCP fails on the resolved port:** almost certainly a firewall rule gap, not a DNS or application problem — check the relevant firewall's rule set for this exact source/destination/port combination before assuming anything else.
- **Both succeed, but the actual application (e.g. `apt`) still fails:** the issue is likely protocol-specific (e.g. the app is trying `http://` against a port-443-only rule) rather than basic connectivity — check what port/scheme the failing request is actually using, not just whether *a* connection to the host works.

## Notes

- Once `curl` is actually installed, `curl -v <url>` gives a much more detailed picture (TLS handshake stages, redirect chains, response headers) — these bare-bones checks are specifically for the window before that's available, or for isolating DNS/TCP from application-layer behavior even after curl exists.

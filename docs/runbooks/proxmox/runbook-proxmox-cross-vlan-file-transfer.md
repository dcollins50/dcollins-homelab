# Runbook: Transfer a File Between Firewall-Isolated Hosts via Proxmox

**Category:** Proxmox
**When to use:** You need to copy a file (a cert, a key, config) from one host to another, but the two hosts sit on VLANs with no firewall rule permitting direct traffic between them — and adding one just to move a single file would be the wrong trade-off (e.g. punching a hole from a public-facing DMZ box into your most protected internal VLAN).

## Why this works

If both source and destination are VMs/LXCs on the same Proxmox host (or cluster), the Proxmox host itself can reach both over the management network, independent of the VLAN firewall rules governing traffic *between the guests*. Routing the copy through the host avoids ever creating a network path between the two guests at all.

## Steps

**1. Pull the file from the source guest onto the Proxmox host** (a plain scp, since the host itself typically has broader reach than either guest):
```bash
# on the Proxmox host
scp user@<source-guest-ip>:<path-to-file> /tmp/<filename>
```

**2. Push it from the host directly into the destination container's filesystem** — this does not go over the network at all; `pct push` writes directly into the container's storage from the host side:
```bash
pct push <destination-vmid> /tmp/<filename> <path-inside-container>
```

**3. Clean up the temp copy on the host:**
```bash
rm /tmp/<filename>
```

**4. Verify on the destination:**
```bash
pct exec <destination-vmid> -- ls -la <path-inside-container>
```

## When *not* to use this pattern

If the transfer needs to happen routinely (not a one-off), this manual host-hop is the wrong long-term answer — it means the file needs to move that way every time it changes. At that point, either the two hosts genuinely need a narrowly-scoped, permanent firewall rule for this specific traffic, or the underlying design (why does an isolated host need this file at all?) is worth revisiting.

## Notes

- `pct push`/`pct pull` only work for LXCs, not full VMs. For a VM on either end, use the Proxmox host's own network reachability to that VM instead (plain `scp` from the host if the host has a route the guests don't).
- This is a security-relevant deliberate choice, not a workaround to feel bad about — it's the correct way to keep two guests from ever needing a direct network path when their only actual requirement is moving a file once.

# Runbook: Primary DNS Failover / Outage Contingency (Pi-hole)

| Field | Value |
| --- | --- |
| Applies to | Pi-hole (Heimdall) |
| Category | DNS / Contingency |
| Author | Daniel Collins |

---

## Open item before this can be finalized

This runbook needs one fact confirmed before it's fully accurate: during a past ~1-2 month cluster shutdown, a separate house Pi reportedly became primary DNS for the personal workstation, with Heimdall's Pi-hole running as secondary during that period. **Is that arrangement still active, fully reverted to Heimdall as sole primary, or something in between?** The steps below are written as a general contingency (what to do if Heimdall's Pi-hole becomes unreachable), which holds either way, but the "Current setup" section needs a real answer filled in rather than an assumption.

---

## When to use this

Use this when Heimdall's Pi-hole becomes unreachable or stops resolving, whether from Heimdall itself going down, the Pi-hole service failing, or a network path issue, and DNS resolution across the homelab (and possibly the wider household network) is affected as a result.

---

## Current setup

*(fill in once the open item above is resolved)*

- Primary DNS for the homelab VLANs: Heimdall's Pi-hole (192.168.100.1)
- Primary DNS for the personal workstation specifically: *unconfirmed, see open item above*
- Secondary/fallback DNS in place: *unconfirmed*

---

## Diagnosing a Pi-hole outage

1. **Is Heimdall itself reachable?**
   ```bash
   ping 192.168.100.1
   ```
   If Heimdall is fully down, this is a Heimdall/hardware issue, not a Pi-hole-specific one, see whatever runbook covers Heimdall's WiFi bridge/WireGuard role for broader recovery.

2. **Is Heimdall up but Pi-hole itself not resolving?**
   ```bash
   dig @192.168.100.1 google.com
   ```
   Timeout or SERVFAIL here with Heimdall otherwise reachable points at the Pi-hole/FTL service specifically, not the whole device.

3. **Check the Pi-hole service directly** (via SSH to Heimdall):
   ```bash
   systemctl status pihole-FTL
   ```

---

## Immediate mitigation while Pi-hole is down

- **If clients are hardcoded to Heimdall as their sole DNS server**, they'll fail to resolve anything until it's back, the fastest mitigation is pointing affected clients at a fallback resolver temporarily (whatever this environment's actual fallback is, per the open item above).
- **VLANs relying on OPNSense's Unbound forwarding to Heimdall** (`homelab.local` resolution) will lose internal hostname resolution specifically, external DNS may still work depending on OPNSense's own fallback configuration, don't assume total DNS failure without checking which layer is actually affected.

---

## Recovery

1. Restart the Pi-hole service if it's the specific thing down:
   ```bash
   systemctl restart pihole-FTL
   ```
2. If Heimdall itself needed a reboot/recovery, confirm Pi-hole comes back up cleanly afterward, don't assume it does just because Heimdall itself is reachable again.
3. **Confirm both internal (`homelab.local`) and external resolution work** post-recovery, not just one or the other, these can fail independently depending on the root cause.
4. If a temporary fallback resolver was put in place for any clients during the outage, revert them back to normal once Pi-hole is confirmed healthy.

---

## Related Documentation

- [Network Architecture](../../network.md)
- [Unbound DNS Forwarding](../opnsense/unbound-dns-forwarding.md)

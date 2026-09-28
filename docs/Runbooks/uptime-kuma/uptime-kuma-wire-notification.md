---
name: uptime-kuma-wire-notification
type: runbook
tool: uptime-kuma
---

Wire an Uptime Kuma monitor's notifications to [[ntfy]] or to a [[soar-host]] (Shuffle) webhook. Covers a real TLS gotcha hit wiring Kuma to Shuffle that cost real debugging time.

## Default notification behavior

Out of the box, Kuma fires on exactly two events per monitor: down (after its configured retry count, not on the first failed check) and back-up. Nothing fires on every successful check. Certificate expiry warnings (21/14/7 days out) are a separate opt-in per monitor, for HTTPS monitors, not automatic.

Applying a notification to "all monitors" means every tracked service (Wazuh, OPNSense, NPM, Vaultwarden, Gitea, Portainer, whatever's being watched) pushes independently on its own up/down transitions. A full node reboot or a `docker compose restart` cycling through several containers can fire a burst of down/up pairs across everything at once, worth considering before blanket-applying a notification to every monitor.

## Wiring to ntfy

ntfy is a plain HTTP POST/PUT endpoint (`curl -d "message" https://ntfy.homelab.example/<topic>`), so Kuma's built-in ntfy notification type (or a generic Webhook/HTTP notification pointed at the topic URL) works directly. Use a scoped, write-only ntfy account/token for the specific topic Kuma publishes to, rather than a shared or admin credential, this homelab's ntfy setup already uses `deny-all` by default with per-topic accounts (see [[ntfy]]).

## Wiring to Shuffle (soar-host)

**The TLS gotcha:** Shuffle's HTTPS webhook endpoint (port 3443) uses its own self-signed cert by default, never fronted with NPM. Kuma's Webhook notification type has no "ignore TLS" toggle (unlike its HTTP *monitor* type, which does have one), so a webhook POST to the HTTPS endpoint fails silently with a `self-signed certificate` error, not something that surfaces clearly in Kuma's UI.

**Fix**: since this is internal-only traffic between two homelab VMs already behind the firewall, use Shuffle's plain HTTP endpoint instead of fighting the cert. In the Kuma notification's Post URL, swap scheme and port only, everything else stays identical:
```
https://<soar-host-ip>:3443/api/v1/hooks/webhook_<id>
```
becomes:
```
http://<soar-host-ip>:3001/api/v1/hooks/webhook_<id>
```
The `webhook_<id>` portion is the unique identifier for that specific trigger node in Shuffle and doesn't change based on transport.

**Before assuming it's still broken after the URL fix**: confirm the firewall rule covers the port actually being used. If the existing rule only permits 3443 (HTTPS) and the fix moved traffic to 3001 (HTTP), the rule needs widening too.

**If it still doesn't land after both fixes**, check what Kuma itself logged when it tried to fire:
```bash
docker logs uptime-kuma --tail 50
```
(container name may differ, `docker ps` first if unsure). Distinguishes "Kuma attempted the call and failed" from "Kuma never attempted it at all", two different problems with different next steps.

## Checking monitor status from automation (e.g. a Shuffle recheck step)

When a workflow needs to ask "is this monitor still down" (the recheck step in a debounce pattern, for example), there are two approaches with a real tradeoff, decided in favor of Kuma's own status over direct-pinging the target:

**Direct-ping the target**: simplest, a plain HTTP GET with no auth or parsing, but only works for monitors that have an actual URL (HTTP-type checks). A TCP port monitor or a Push-type monitor (nothing to ping, the monitored thing pushes heartbeats *to* Kuma) has no target to hit this way.

**Query Kuma's own status via its `/metrics` endpoint**: works uniformly for every monitor type, since it asks Kuma what it currently thinks rather than touching the target directly. The tradeoff is parsing complexity: `/metrics` returns plain Prometheus-format text (one line per monitor per metric, e.g. `monitor_status{monitor_name="test-debounce-harness",monitor_type="push"} 0`), not clean JSON, so consuming it means an HTTP request with Basic Auth (Kuma API key as the password, username ignored) plus a regex/text-extraction step to find the specific monitor's line and pull the trailing 0/1 value.

Chosen here because the actual monitor set is a mix of types, including Push-type monitors with nothing pingable, so direct-ping can't cover every monitor uniformly. If every monitor in a given deployment is HTTP-type, direct-ping is meaningfully simpler and worth reconsidering.

## Related
- [[ntfy]]
- [[soar-host]]
- [[sop-soc-alert-pipeline]]

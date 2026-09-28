---
name: iris-test-alert-api
type: runbook
tool: dfir-iris
---

Confirm [[pve-iris]]'s (DFIR-IRIS) alert-creation API is actually reachable and working, end to end, with a manual test call. This is the verification step before wiring any automation (Shuffle, Wazuh active response, etc.) into IRIS, prove the endpoint works standalone first. Written up from the working call that proved this out in September 2026.

## Why API, not webhooks

Decided in favor of the REST API over IRIS's webhook module: the API is part of the core `iris-web` repo (LGPL3, actively maintained), while the webhook module is a separate add-on with old unresolved issues and limited maintainer attention. Functionally equivalent for this use case either way, since something else (Shuffle, a script) still has to be watching for the trigger event and calling out, IRIS doesn't push anything on its own via the API. The API is just the more reliably maintained path to do that call against.

## Prerequisites

- IRIS reachable at its NPM-fronted URL (`https://iris.homelab.local`) with a properly signed cert
- An IRIS API key: log into IRIS, click your username top-right → **My Settings**, the key is shown there

## Procedure

**Endpoint is `/alerts/add`, not `/api/v1/alerts`.** The first attempt used `/api/v1/alerts` and failed; `/alerts/add` is correct for this IRIS version.

**Required fields.** A minimal payload with just title/description/source/severity is rejected, `alert_status_id` and `alert_customer_id` are also required. On a fresh install, `alert_customer_id: 1` is the default customer ("IrisInitialClient") and `alert_status_id: 1` is "New"/unassigned. If `1` doesn't resolve correctly on a given instance, confirm the real IDs under **Advanced → Customers** in the UI first.

**From the Windows workstation (PowerShell):**
```powershell
$headers = @{
    "Authorization" = "Bearer YOUR_API_KEY"
    "Content-Type" = "application/json"
}

$body = @{
    alert_title = "Test alert from curl"
    alert_description = "Manual API test"
    alert_source = "manual-test"
    alert_severity_id = 2
    alert_status_id = 1
    alert_customer_id = 1
} | ConvertTo-Json

Invoke-RestMethod -Uri "https://iris.homelab.local/alerts/add" -Method Post -Headers $headers -Body $body
```
A successful response looks like `status: success` with alert data echoed back (severity, status, customer, classification, owner, iocs, assets, etc.).

**From bash** (e.g. SSH'd into a Linux host instead of the Windows workstation):
```bash
curl -k -X POST https://iris.homelab.local/alerts/add \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "alert_title": "Test alert from curl",
    "alert_description": "Manual API test",
    "alert_source": "manual-test",
    "alert_severity_id": 2,
    "alert_status_id": 1,
    "alert_customer_id": 1
  }'
```
`-k` skips cert verification, drop it if the workstation/host already trusts the internal PKI chain.

**Confirm in the UI**, not just the API response: check the IRIS **Alerts** view and verify the test alert is actually sitting there. The API returning success and the alert actually persisting are two different things worth checking separately.

## Notes

- Windows PowerShell 5.1 does not have `-SkipCertificateCheck` on `Invoke-RestMethod` (that flag is PowerShell 7+ only). If the cert is properly signed and trusted, this isn't needed anyway.
- Automated triggers (Shuffle, Wazuh active response) should create IRIS **Alerts**, not **Cases**, to avoid cluttering the case workload with unreviewed automated noise. Cases are for confirmed incidents a human has escalated to.

## Related
- [[pve-iris]]
- [[soar-host]]
- [[sop-soc-alert-pipeline]]

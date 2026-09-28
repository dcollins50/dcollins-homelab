---
name: wazuh-check-agent-status
type: runbook
tool: wazuh
---

Check which Wazuh agents are connected and reporting to [[wazuh-manager]]. Extracted from the troubleshooting section of `Projects/soc-stack-buildout/soc-phase1-baseline.md`.

## When to use this

Alert counts look unexpectedly low, a specific agent seems to have gone quiet, or before trusting a "zero hits" result during rule confirmation (see [[wazuh-restart-agent-confirm-scan]]) — a disconnected agent won't produce alerts either way, so a clean query result could mean the fix worked or could mean the agent isn't reporting at all.

## Procedure

On wazuh-manager:
```
/var/ossec/bin/agent_control -la
```

Lists all enrolled agents with their connection status (Active, Disconnected, Never connected, Pending). Any agent showing Disconnected when it should be Active needs its own investigation, service status on that host, network path back to wazuh-manager on TCP/UDP 1514–1515, etc.

## Related
- [[wazuh-manager]]
- [[wazuh-restart-agent-confirm-scan]]

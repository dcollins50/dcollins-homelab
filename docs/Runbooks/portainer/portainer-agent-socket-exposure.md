---
name: portainer-agent-socket-exposure
type: runbook
tool: portainer
---

Check for, and fix, a Portainer agent unnecessarily publishing the Docker socket to the host network. Found and fixed on [[services-host]] during the Sep 13, 2026 incident investigation.

## The problem

A Portainer agent container running with `-p 9001:9001` publishes its port to the host's network interface, meaning anything that can reach that host on the LAN can talk to the Docker socket the agent proxies, not just Portainer's own server container. This is broader exposure than needed: Portainer's server only needs to reach the agent over the internal Docker network, not from outside the host.

## How to check

```bash
docker ps --filter "name=portainer" --format "table {{.Names}}\t{{.Ports}}"
```
If the agent's ports column shows `0.0.0.0:9001->9001/tcp` (published to all interfaces) rather than nothing or an internal-only binding, it's exposed.

## Fix

Recreate the agent container without the host port publish, and point Portainer's environment endpoint at the agent via Docker-internal DNS (the container name) instead of the host's IP and published port.

1. Remove the existing agent container (adjust name if different):
   ```bash
   docker stop portainer_agent
   docker rm portainer_agent
   ```

2. Recreate it without `-p 9001:9001`:
   ```bash
   docker run -d \
     --name portainer_agent \
     --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v /var/lib/docker/volumes:/var/lib/docker/volumes \
     portainer/agent:latest
   ```
   Same image and volume mounts as before, just no `-p` flag.

3. In the Portainer server UI, edit the environment/endpoint that pointed at `<host-ip>:9001` and change it to the agent's Docker-internal DNS name and port (e.g. `tcp://portainer_agent:9001`), works as long as the Portainer server and agent share a Docker network (default bridge is usually sufficient on a single host).

4. Confirm the endpoint still shows connected/healthy in the Portainer UI after the change.

## Related
- [[services-host]]
- [[incident-2026-09-13-elastic-log-shipping-outage]] (found during this investigation, unrelated root cause)

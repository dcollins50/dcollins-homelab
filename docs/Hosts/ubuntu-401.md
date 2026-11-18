---
name: ubuntu-401
type: vm
vmid: 401
node: pve-services
ip: 10.0.30.x
vlan: VLAN30
status: live
role: General services
---

**Login:** `[admin-user]`, password auth (SSH key not yet copied as of Sep 25, 2026).

General-purpose Ubuntu VM on the Trust Infrastructure VLAN. Runs a Wazuh agent.

IP 10.0.30.x/24, confirmed in [[incident-2026-01-24-vlan-connectivity-fixes]] and [[soc-phase2.5-midweek-state]], and by Daniel on Sep 24, 2026.

## Update Oct 31, 2026: resolver moved to Unbound
DNS now points at 10.0.30.x (Unbound on the VLAN30 gateway). Before: 192.168.100.x (Pi-hole) only. Set in `/etc/netplan/50-cloud-init.yaml` (`nameservers: addresses: [10.0.30.x]`, interface `enp6s18`). Backup at `~/50-cloud-init.yaml.bak-2026-10-31` on the VM. cloud-init is disabled here, so the file is not regenerated. The file still uses the deprecated `gateway4` key (netplan warns on every generate); not changed. Applied with `netplan try`, accepted at 10:06. The shell hostname is `[admin-user]-Standard-PC-Q35-ICH9-2009`, not `ubuntu-401`. Its resolver stub handles `.local` names itself and returns SERVFAIL, so test with `.internal` names. Part of [[dns-phase1-consolidation]]. Verified at 10:07: `resolvectl status enp6s18` shows `DNS Servers: 10.0.30.x` and `getent hosts stepca.homelab.internal` returns 10.0.30.x.

## Related
- [[pve-services]]
- [[VLAN30]]

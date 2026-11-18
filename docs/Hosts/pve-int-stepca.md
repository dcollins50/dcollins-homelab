---
name: pve-int-stepca
type: lxc
vmid: 511
node: pve-services
ip: 10.0.30.x
vlan: VLAN30
status: live
role: step-ca (Docker-in-LXC), planned Intermediate CA replacement
---

**Login:** `[admin-user]`, password auth. Normally accessed via `pct enter` from [[pve-services]] rather than direct SSH.

Native Debian 13 LXC running Docker, with step-ca as a container (nesting enabled). Built Sep 19, 2026; Docker confirmed working same day (hello-world ran).

Local user "[admin-user]" added (LXC previously had no non-root user).

**Migration approach:** import [[pve-ca-intermediate]]'s existing cert/key rather than generate a fresh root, to preserve trust continuity. Real root cert, intermediate cert, and intermediate key have been imported into the Docker volume, overwriting the throwaway bootstrap PKI — verified via matching SHA256 root fingerprint against [[pve-ca-root]]'s actual cert.

**Authentik OIDC provisioner:** registered and confirmed via `step ca provisioner list` (client ID, config endpoint, enableSSHCA true). Required: enabling step-ca remote admin management, an OPNSense Unbound host override for stepca.homelab.local (no DNS record existed initially), a DEV VLAN firewall rule (pve_int_stepca alias → internal_npm alias, port 443), and importing the homelab root CA cert into the container's trust store.

**Next step:** the OIDC provisioner is registered, this is for SSH bastion cert issuance — the bastion ([[pve-bastion]]) was built Sep 21, 2026, but SSH does not use the certs it issues yet, because sshd trust on the bastion is not configured.

## Related
- [[pve-services]]
- [[pve-ca-intermediate]]
- [[pve-ca-root]]
- [[authentik]]
- [[VLAN30]]

## Update Oct 3, 2026: SSH CA configured
step-ca had no SSH CA until today (`step ssh config --roots` returned "not configured with SSH support"). Added `ssh_host_ca_key` and `ssh_user_ca_key` in the container's `/home/step/secrets` (public halves in `/home/step/certs`), and an `ssh` block in `config/ca.json` after `federatedRoots`. Backup of the old file: `config/ca.json.bak-2026-10-03`. The Authentik provisioner's client secret was re-entered because the token exchange failed with `invalid_client`. The Authentik provider is `step-ca` with its own application `step-ca` and a Regex redirect entry for the loopback address. See [[session-2026-10-03-stepca-ssh-ca]] and [[bastion-ssh-certificate-login]]. Open: backup of these new keys, pinning certificate lifetimes, rotating the admin password, and a leftover `secrets/root_ca_key` to investigate.

## Update Oct 17, 2026: container was down for a week, restart policy set
The `step-ca` container exited (255) on 2026-10-10 at 02:51 UTC with no error in its log and stayed down, including through 511's reboot on Oct 17, because its restart policy was `no`. Cause of the stop not found. Started by hand, then the restart policy was set to `unless-stopped`. Check with `sudo pct exec 511 -- docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' step-ca` from pve-services. Proven the same day: step-ca answered `/health` after `pct reboot 511` with no manual start. Details in [[session-2026-10-17-stepca-remote-tunnel]].

## Update Oct 29, 2026: same intermediate key as VM 501 confirmed
The intermediate cert's public key hash matches [[pve-ca-intermediate]]'s (checked from pve-services with `sudo pct exec 511 -- docker exec step-ca cat /home/step/certs/intermediate_ca.crt | openssl x509 -noout -pubkey | sha256sum`). Same key pair on both machines. pve-int-stepca has no `sudo`, so run commands for this container from pve-services with `pct exec 511`.
Resolver: `/etc/resolv.conf` is PVE-managed, `nameserver 10.0.30.x` (Unbound on the VLAN30 gateway), `search homelab.local`, no systemd-resolved (checked Oct 29, 2026). Unbound answers `stepca.homelab.local` as 10.0.30.x; `stepca.homelab.internal` has no record. Whether the step-ca container uses the same resolver is not checked.

## Update Nov 9, 2026: new intermediate key installed
step-ca now signs with a new intermediate (RSA 4096, CN `Homelab Intermediate CA` without the leading space, valid to Nov 6, 2036) signed by the offline root. The old key is no longer used by step-ca, so it exists only on [[pve-ca-intermediate]] and in `.old-2026-11-09` files on the volume. Proven with an ACME renewal on [[ubuntu-401]]. Encrypted pre-change backup of the volume: `/root/step-backup-2026-11-09.tgz.enc` here and `D:\step-backup-2026-11-09.tgz.enc` on the workstation. The new key file must be `1000:1000`; `cp` over an existing file in the BusyBox container does not keep ownership. See [[session-2026-11-09-stepca-intermediate-rotation]].

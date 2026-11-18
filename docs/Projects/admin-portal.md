---
name: admin-portal
type: project
status: planned
---

A browser page that lists the web UIs of the homelab as tiles, in the style of a MyApps launcher, so Daniel stops keeping about 14 bookmarks. Explored Oct 3, 2026. Nothing is built. Lines marked stated were said by Daniel. Lines marked proposed are Claude's suggestions and are not agreed.

## Stated requirements
- Reached through the bastion ([[pve-bastion]]), which is the only route to the portal host. The path between the bastion and the portal host is heavily locked down, and each web UI keeps its own login.
- The bastion's web entry needs its own authentication. Logging in to the tile page is a separate login with its own MFA.
- A container between the bastion and the LAN is acceptable as the host.
- Reached by a name (FQDN), not by Mesh IP. It should be easy for other organizations to adopt without dropping defensive layers.
- Authentik forward auth ([[authentik]]) should be available only to a browser that holds a step-ca client certificate ([[pve-int-stepca]]).
- Client certificates rotate automatically every 24 hours.
- Devices: the workstation (Brave now, Firefox or Mullvad Browser later) and a phone. The workstation sleeps at night. Wants to explore MDM enrollment for the phone.
- Wants a hard Cloudflare check on the certificate in addition to the step-ca check. Paying $7 a month for Cloudflare is acceptable.
- The workstation hostname is the Windows default and Daniel is not sure he wants to keep it. He may rename it.

## Proposed design
- Private road: the name resolves only for WARP-enrolled devices. Traffic rides the WARP tunnel to a proxy on the bastion, which requires the client certificate, then Authentik forward auth, then the portal. This keeps TLS end to end, so the proxy can check the certificate itself.
- Cloudflare check on the private road: a WARP client certificate posture check, plus a Gateway network policy that blocks traffic to the bastion from devices that have not passed it. Whether a Gateway policy can gate a Mesh node is not confirmed.
- Cloudflare pins the direct signer of the client certificate, not the root. A dedicated signing certificate for client certificates is therefore cleaner. It needs a new CA certificate and a second step-ca instance, because one step-ca instance issues from one CA only. The second instance can be another container in [[pve-int-stepca]], not a new LXC. The new CA certificate can be signed by the existing online intermediate ([[pve-ca-intermediate]]) if its path length allows a sub-CA, otherwise by the offline root ([[pve-ca-root]]).
- Workstation: `step ca renew --daemon` plus a post-renewal import into the Windows certificate store, which Brave reads.
- Phone: Cloudflare's client certificate check does not support iOS or Android. Option A is the private road with identity and WARP enrollment checks at Cloudflare and the certificate checked only by the bastion's proxy. Option B is a public name with Cloudflare Access client certificate checks, which needs the pay-as-you-go plan.

## Findings checked against docs on Oct 3, 2026
- Cloudflare's own client certificate checks against a custom CA are Enterprise only (bring your own CA) or Enterprise and pay-as-you-go (Access). Not on the free plan.
- A public Cloudflare Tunnel name ends TLS at Cloudflare, so the origin cannot see the browser's certificate.
- The WARP client certificate posture check is on all plans, supports Windows, macOS and Linux, and does not validate the chain. It can match CN, SAN and EKU, including hostname and serial number variables.
- Cloudflare Mesh traffic is layer 3 and 4, and clients can reach a node by Mesh IP on any port.
- step-ca certificates last 24 hours by default. `step ca renew --daemon` renews at about two thirds of the lifetime. Renewal after expiry is a provisioner option, and Smallstep warns it adds risk.
- step-ca's SCEP provisioner needs an RSA intermediate. Device attestation for phones is mostly in Smallstep's hosted product.
- Brave on Windows uses the Windows certificate store. Firefox uses its own store unless `security.osclientcerts.autoload` is set.

## Open decisions
1. Name resolution: a public DNS record pointing at the bastion's private address, or a Cloudflare resolver rule.
2. Signer: **decided Oct 3, 2026: dedicated signing certificate** for browser certificates, see [[admin-portal-browser-certificate]]. Not built.
3. Phone path: **decided Oct 3, 2026: no phone for now.** The phone path (A or B) stays undecided.
4. Long power-off policy for the workstation: **decided Oct 3, 2026: log in again.** An expired browser certificate means a fresh login to get a new one. No longer lifetime and no renewal after expiry.
5. Hostname binding for the workstation certificate: **decided Oct 3, 2026: rename the workstation first, then bind the new name into the certificate.** The new hostname is not chosen yet.
6. Phone rotation: deferred with the phone path. iPhone or Android is not known yet. MDM does not fix the gap in Cloudflare's phone support.
7. Which host runs the portal. Daniel gave a Proxmox node summary for [[pve-services]] on Oct 3, 2026 (about 15:13 local): RAM 6.13 GiB used of 15.50 GiB (39.52%), root disk 9.67 GiB of 93.93 GiB (10.30%). That is one snapshot of current use. Memory assigned to each guest, ballooning and limits were not checked, so the host choice is still open and RAM alone does not decide it. The network placement (which VLAN, and the lockdown between the bastion and the portal host) matters more than RAM.
8. The bastion was disconnected until Oct 3, 2026. WARP now stays connected with IPv6 enabled on the WARP interface only, see [[session-2026-10-03-bastion-warp-ipv6]]. The workstation now reaches the bastion over the Mesh (ping and SSH tested Oct 3, see [[session-2026-10-03-bastion-warp-ipv6]]). Still untested: the Mesh access policies themselves, and the phone path.

9. MFA shape: **decided Oct 3, 2026: two separate Authentik MFA prompts**, one at the bastion's web entry and one at the tile page. This only holds if Authentik forces a fresh prompt each time. The step-ca login reused the existing Authentik session with no prompt on Oct 3, 2026 ([[bastion-ssh-certificate-login]]), so the same would happen here. The Authentik setting to force it is not verified. This is the next thing to fix before the portal is built.

## Not verified
- Whether a Gateway network policy can gate traffic to a Mesh node.
- How often the posture check re-runs, and what happens when a 24-hour certificate expires.
- Whether step-ca's default template includes client authentication usage.
- How the renewal daemon behaves after sleep, and whether it runs cleanly on Windows.
- Whether Mullvad Browser honors the Firefox OS certificate setting.
- The $7 per user figure comes from a third-party source. Confirm on Cloudflare's pricing page.
- Whether the existing intermediate allows a sub-CA, and whether its key is RSA.

## Related
- [[pve-bastion]]
- [[pve-int-stepca]]
- [[authentik]]
- [[remote-access-path-order]]
- [[dns-architecture]]
- [[sop-ssh-bastion-build]]

---
name: admin-portal-browser-certificate
type: decision
status: live
---

## Decision
Browser access to the admin portal ([[admin-portal]]) requires an X.509 client certificate issued by step-ca ([[pve-int-stepca]]). Decisions made by Daniel on Oct 3, 2026. Nothing is built yet.

- **Signer:** a dedicated signing certificate used only for browser certificates, run as a second step-ca instance (a second container on the same host, not a new LXC). Cloudflare pins the direct signer of a client certificate, so this keeps server certificates out of scope. The new CA certificate is signed by the existing intermediate ([[pve-ca-intermediate]]) if its path length allows a sub-CA, otherwise by the offline root ([[pve-ca-root]]). Neither has been checked.
- **Lifetime and expiry:** 24 hours, as stated earlier. After a long power-off the certificate is expired and Daniel logs in again to get a new one. No longer lifetime and no renewal after expiry.
- **Devices:** the workstation only for now. No phone. The phone path stays undecided.
- **Hostname:** rename the workstation first (still the Windows default name), then put the new hostname into the certificate. The new name is not chosen yet.
- **MFA:** two separate Authentik MFA prompts in the browser path, one at the bastion's web entry and one at the tile page.

## Why
Daniel wants each layer to stand on its own and said he wants layering as far as it can go. A dedicated signer lets the bastion's proxy accept only browser certificates, instead of anything the intermediate has issued with client authentication usage.

## Not the same as the SSH certificate
The SSH certificate from [[bastion-ssh-certificate-login]] only works with `ssh`. A browser needs an X.509 client certificate. step-ca can issue both, but this is a separate issuance path with its own template, its own import into the Windows certificate store (Brave reads it) and its own rotation.

## Open problems
- **MFA re-prompt:** the step-ca login reused the Authentik session with no MFA prompt on Oct 3, 2026. Two separate prompts only exist if Authentik forces one each time. The Authentik setting is not verified. Fix and test it first.
  Checked Authentik's docs on Oct 3, 2026 (authenticator validation stage, example flows, security hardening). Found: the Authenticator Validation stage has a "Last validation threshold" option that skips the prompt if a compatible device was used within a time window, and "Not configured action". The docs describe it for authentication flows after the login stages and do not say whether it can re-prompt an already signed-in user from an authorization flow. None of the seven example flows does that. Hardening docs say session duration defaults to 0 seconds, which ends the session when the browser closes. So the fix is not confirmed by the docs. The way to find out is a test on the `step-ca` provider's authorization flow only. Not done.
- **Issuance trust:** if browser certificates are issued through the same Authentik login as the SSH certificate, a compromised Authentik can mint one, and then the certificate layer does not stand alone. How to issue them without that dependency is not decided.
- **The bastion becomes a web host.** Today it accepts only SSH on the WARP interface. A browser path needs a web proxy on it, a firewall rule and a Gateway policy for port 443. The SSH tunnel alternative keeps the bastion free of web services but is not "type a domain and it loads".
- **Name resolution:** a public DNS record or a Cloudflare resolver rule. Not decided.
- **Phone rotation and MDM:** deferred.
- **Portal host:** which VLAN and which host. RAM on [[pve-services]] is not the constraint (6.13 of 15.50 GiB used on Oct 3, 2026).

## Related
- [[admin-portal]]
- [[bastion-ssh-certificate-login]]
- [[pve-int-stepca]]
- [[authentik]]

# Runbook: Import an Existing CA into step-ca (Docker-in-LXC)

**Category:** Internal PKI
**When to use:** Standing up step-ca to take over signing duties from an existing OpenSSL (or other) CA, preserving the existing trust chain rather than generating a new root.

## Prerequisites
- An unprivileged LXC with Docker installed and nesting enabled (`pct set <vmid> --features nesting=1`), or a privileged LXC.
- The existing root cert, intermediate cert, and intermediate private key, plus its OpenSSL passphrase.
- Outbound firewall access to Cloudflare/Fastly edge ranges for package installs, and to the CA's own hostname for DNS/NTP as applicable.

## Steps

1. **Generate the boilerplate PKI**, setting the real admin password and enabling remote management up front — do not leave these to defaults, they cannot be safely recovered later if the auto-generated password file gets overwritten:
   ```bash
   docker run -d -v step:/home/step \
     -p 9000:9000 \
     -e "DOCKER_STEPCA_INIT_NAME=<Issuer Name — match existing intermediate cert's CN exactly>" \
     -e "DOCKER_STEPCA_INIT_DNS_NAMES=<hostname>,<ip>" \
     -e "DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true" \
     -e "DOCKER_STEPCA_INIT_PASSWORD=<a real password you will save immediately>" \
     smallstep/step-ca
   ```
   Save the admin username/password pair that prints in the container logs (`docker logs <container>`) to your password manager **immediately** — it is shown only once.

2. **Confirm the throwaway PKI initialized cleanly**: `docker logs <container>` should show `Serving HTTPS on :9000` with no errors.

3. **Copy the real cert/key files onto the LXC host.** If the source CA lives on a different, firewall-isolated network segment, route the copy through the Proxmox host rather than opening a new cross-VLAN firewall rule:
   ```bash
   # on the Proxmox host
   scp user@<source-ca-ip>:<path>/root-ca.crt /tmp/
   scp user@<source-ca-ip>:<path>/intermediate-ca.crt /tmp/
   scp user@<source-ca-ip>:<path>/intermediate-ca.key /tmp/
   pct push <vmid> /tmp/root-ca.crt /root/root-ca.crt
   pct push <vmid> /tmp/intermediate-ca.crt /root/intermediate-ca.crt
   pct push <vmid> /tmp/intermediate-ca.key /root/intermediate-ca.key
   rm /tmp/root-ca.crt /tmp/intermediate-ca.crt /tmp/intermediate-ca.key
   ```

4. **Stop the container and import the files**, overwriting the throwaway ones:
   ```bash
   docker stop <container>
   docker cp root-ca.crt <container>:/home/step/certs/root_ca.crt
   docker cp intermediate-ca.crt <container>:/home/step/certs/intermediate_ca.crt
   docker cp intermediate-ca.key <container>:/home/step/secrets/intermediate_ca_key
   ```

5. **Fix ownership on the imported key** — `docker cp` writes as root, but step-ca's process runs as UID 1000:
   ```bash
   docker run --rm -v step:/home/step alpine chown 1000:1000 /home/step/secrets/intermediate_ca_key
   ```

6. **Set the password file to the real intermediate key passphrase** (this is a *different* value from the admin password saved in step 1 — this one decrypts the signing key at boot, that one authenticates admin API calls):
   ```bash
   docker run --rm -v step:/home/step alpine sh -c 'echo -n "<real OpenSSL passphrase>" > /home/step/secrets/password'
   ```

7. **Restart and verify:**
   ```bash
   docker start <container>
   docker logs <container>
   ```
   Should show a clean start with no permission or decryption errors, and a new X.509 Root Fingerprint matching the real root cert (verify with `openssl x509 -in root-ca.crt -noout -fingerprint -sha256` on the source CA).

## Common failures

| Symptom | Cause | Fix |
|---|---|---|
| `permission denied` reading intermediate_ca_key | `docker cp` wrote the file as root | Step 5 above |
| `x509: decryption password incorrect` | password file still has the throwaway/auto-generated value | Step 6 above |
| `step ca provisioner add` fails with local `ca.json` file-not-found error | Remote management not enabled, or CLI has no bootstrapped trust | Confirm `DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true` was set at init; run `step ca bootstrap --ca-url <url> --fingerprint <fp>` first |
| Admin login (`step ca admin add`, `step ca provisioner add`) fails no matter what password is tried | The admin password and the intermediate key passphrase are two different secrets — confirm which one is actually needed at the specific prompt | Re-read the prompt context; do not assume they're interchangeable |
| `apt`/package installs hang or time out | Origin is behind a CDN (Fastly, Cloudflare) whose edge IP doesn't match a narrowly-scoped firewall alias | Widen the alias to the provider's full published IP range, not individual resolved IPs |

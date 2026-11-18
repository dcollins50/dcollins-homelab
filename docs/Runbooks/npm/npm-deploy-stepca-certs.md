---
name: npm-deploy-stepca-certs
type: runbook
tool: npm
---

# Runbook: Get step-ca certificates into internal NPM

**When to use:** Issuing and renewing `*.homelab.internal` certificates for proxy hosts on the internal NPM in [[services-host]]. NPM cannot request certificates from [[pve-int-stepca]] itself (no custom ACME server option) and cannot update a custom certificate in place (PR #5959 still open as of Nov 9, 2026), so certbot on services-host gets the certificate and a script uploads it as a new custom entry, points the proxy host at it, and deletes the old entry. Built and proven Nov 9, 2026 on `uptime.homelab.internal`, see [[session-2026-11-09-stepca-intermediate-rotation]].

## How it fits together
- step-ca checks a name by visiting `http://<name>/` on port 80, which lands on NPM. A file at `/home/admin/docker/nginx-proxy-manager/data/nginx/custom/server_proxy.conf` (NPM includes it in every proxy host) forwards `/.well-known/acme-challenge/` to certbot's temporary listener on services-host port 8899. Port 8888 is taken by another container.
- Contents of that file: `location ^~ /.well-known/acme-challenge/ { proxy_pass http://10.0.20.x:8899; }`
- Firewall rules in OPNsense (both narrow, logged): Services interface, TCP, source `internal_npm`, destination `pve_int_stepca`, port 9000 (certbot to the CA). DEV interface, TCP, source `pve_int_stepca`, destination `internal_npm`, port 80 (the check). Entering the source and destination backwards cost about 40 minutes on Nov 9, 2026.
- services-host trusts the root: `/usr/local/share/ca-certificates/homelab-root.crt`, fingerprint-checked against `0201efdc...77e45` before install.
- The ACME provisioner has `forceCN` on, because NPM cannot read certificates with an empty subject.
- NPM is pinned to `2.16.0` in the compose file (upgraded from 2.13.5 Nov 9, 2026 for CVE-2026-40519).

## Issue a certificate for a new name
1. Make sure Unbound has the `.homelab.internal` name pointing at 10.0.20.x and an NPM proxy host exists for it (the script needs the host to attach to).
2. On services-host: `sudo certbot certonly --standalone --http-01-port 8899 --non-interactive --agree-tos --register-unsafely-without-email --server https://stepca.homelab.internal:9000/acme/acme/directory -d <name>`
3. `sudo /usr/local/sbin/npm-cert-deploy <name>`. It prints `OK <name>: proxy host N now uses cert M`.
4. Check: `curl -s -o /dev/null -w '%{http_code} verify=%{ssl_verify_result}\n' https://<name>/` gives a normal code and `verify=0`. Wait a few seconds after the script, NPM reloads asynchronously (the first check failed when run in the same second).

## Renewal
`/etc/letsencrypt/renewal-hooks/deploy/npm-cert-deploy` runs `npm-cert-deploy` with the lineage name after every successful renewal, driven by the normal certbot timer. Leaf lifetime is 60 days. Proven with `certbot renew --cert-name uptime.homelab.internal --force-renewal` against a host with Force SSL on (cert 24 replaced by 25, old one deleted).

## Credentials
`/root/.npm-api.env` (mode 600) holds `NPM_USER` and `NPM_PASS`. Never put them in the vault or in chat. A dedicated NPM user limited to certificates and proxy hosts is better than the admin login.

## The script, `/usr/local/sbin/npm-cert-deploy` (mode 700)
```bash
#!/bin/bash
# npm-cert-deploy NAME: put the certbot cert for NAME into NPM and point its proxy host at it
set -euo pipefail
NAME="${1:?usage: npm-cert-deploy NAME}"; export NAME
API=http://127.0.0.1:81/api
LIVE=/etc/letsencrypt/live/$NAME
set -a; . /root/.npm-api.env; set +a
J() { python3 -c "import sys,json;d=json.load(sys.stdin);$1"; }
C() { curl -s --fail-with-body "$@"; }
TOKEN=$(C -X POST $API/tokens -H 'Content-Type: application/json' \
  -d "$(python3 -c 'import json,os;print(json.dumps({"identity":os.environ["NPM_USER"],"secret":os.environ["NPM_PASS"]}))')" | J 'print(d["token"])')
A=(-H "Authorization: Bearer $TOKEN")
OLD=$(C "${A[@]}" $API/nginx/certificates | J 'import os;print(" ".join(str(c["id"]) for c in d if os.environ["NAME"] in c["domain_names"] and c["provider"]=="other"))')
C "${A[@]}" -X POST $API/nginx/certificates/validate -F certificate=@$LIVE/fullchain.pem -F certificate_key=@$LIVE/privkey.pem >/dev/null
NEW=$(C "${A[@]}" -X POST $API/nginx/certificates -H 'Content-Type: application/json' \
  -d "{\"provider\":\"other\",\"nice_name\":\"$NAME $(date +%F)\",\"domain_names\":[\"$NAME\"]}" | J 'print(d["id"])')
C "${A[@]}" -X POST $API/nginx/certificates/$NEW/upload -F certificate=@$LIVE/fullchain.pem -F certificate_key=@$LIVE/privkey.pem >/dev/null
HID=$(C "${A[@]}" $API/nginx/proxy-hosts | J 'import os;print(next((h["id"] for h in d if os.environ["NAME"] in h["domain_names"]),""))')
[ -n "$HID" ] || { echo "no proxy host for $NAME; cert $NEW uploaded but not attached"; exit 1; }
C "${A[@]}" -X PUT $API/nginx/proxy-hosts/$HID -H 'Content-Type: application/json' \
  -d "{\"certificate_id\":$NEW,\"ssl_forced\":true,\"http2_support\":true}" >/dev/null
for o in $OLD; do C "${A[@]}" -X DELETE $API/nginx/certificates/$o >/dev/null || echo "could not delete old cert $o"; done
echo "OK $NAME: proxy host $HID now uses cert $NEW"
```

Deploy hook `/etc/letsencrypt/renewal-hooks/deploy/npm-cert-deploy` (mode 755):
```bash
#!/bin/bash
/usr/local/sbin/npm-cert-deploy "$(basename "$RENEWED_LINEAGE")"
```

## Known limits
- The script sends a partial `PUT` to the proxy host (`certificate_id`, `ssl_forced`, `http2_support`). It worked on the uptime host without wiping its forward settings. It forces SSL and HTTP/2 on every host it touches.
- NPM has no in-place update, so each renewal creates a new entry and deletes the old one. If the script fails after the upload, the new entry is left unattached in NPM's Certificates tab.
- Not yet proven for the 10 other names. A renewal failure shows up as a certbot error in the timer's journal (`journalctl -u certbot`), and Uptime Kuma does not monitor certificate expiry yet.

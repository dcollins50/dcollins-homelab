---
name: vaultwarden-configure-fail2ban
type: runbook
tool: vaultwarden
---

Ban IPs that repeatedly fail Vaultwarden logins, using fail2ban on [[services-host]]. Not set up yet as of Oct 3, 2026. Needs file logging first, see [[vaultwarden-enable-file-logging]].

## Prerequisite that decides if this works
Vaultwarden must log the real client IP. Behind the internal NPM it will log the proxy address unless `X-Real-IP` is forwarded, see [[vaultwarden-npm-proxy-host]]. If the log shows only the proxy address, fail2ban would ban the proxy and lock everyone out.

## Steps
1. Install fail2ban on the host (not in the container): `sudo apt install fail2ban`.

2. Confirm a failed login produces a line like this in the log file:
   ```
   [YYYY-MM-DD hh:mm:ss][vaultwarden::api::identity][ERROR] Username or password is incorrect. Try again. IP: x.x.x.x. Username: someone@example.com.
   ```

3. Create the filter, `/etc/fail2ban/filter.d/vaultwarden.local`:
   ```bash
   sudo tee /etc/fail2ban/filter.d/vaultwarden.local > /dev/null <<'F2B'
   [INCLUDES]
   before = common.conf

   [Definition]
   failregex = ^.*?Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
   ignoreregex =
   F2B
   ```
   If fail2ban reports a missing "host" group, use `<HOST>` instead of `<ADDR>`.

4. Create the jail, `/etc/fail2ban/jail.d/vaultwarden.local`. Because Vaultwarden runs in Docker, add `chain = FORWARD`, since Docker traffic uses the FORWARD chain:
   ```bash
   sudo tee /etc/fail2ban/jail.d/vaultwarden.local > /dev/null <<'F2B'
   [vaultwarden]
   enabled = true
   port = 80,443,8080
   filter = vaultwarden
   banaction = %(banaction_allports)s
   logpath = <FILL IN: host path of vaultwarden.log>
   maxretry = 3
   bantime = 14400
   findtime = 14400
   chain = FORWARD
   F2B
   ```
   The official example lists ports `80,443,8081`. Here the published port is `8080`, so it was changed. Check against the real setup.

5. Restart and check:
   ```bash
   sudo systemctl restart fail2ban
   sudo fail2ban-client status vaultwarden
   ```

6. Test from a spare device or throwaway IP, with three bad logins, and confirm the ban. Do not test from the workstation you manage it from. Unban with `sudo fail2ban-client set vaultwarden unbanip <ip>`.

7. Add your own addresses to `ignoreip` in the jail so you cannot ban yourself.

## Related
- [[vaultwarden-enable-file-logging]]
- [[vaultwarden-harden-container]]
- [[vaultwarden]]

Source: https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup

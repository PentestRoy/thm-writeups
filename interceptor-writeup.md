# Interceptor — TryHackMe Writeup

> **Room:** [Interceptor](https://tryhackme.com/room/interceptor) · **Difficulty:** Medium
> **Category:** Web / Burp (response tampering → RCE)
> **Author of writeup:** 0xnyx
> **Goal:** Intercept the "MediaHub" login flow to reach the admin area, then escalate to a shell.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/interceptor), **no flags are included** — only the method.
>
> ⚠️ CONFIRM: recon (SQLi, extra fields, phpMyAdmin, `.git`/backup leak, password brute) all dead-ended on this box — the intended path is Burp-based. Fill in the exact Q2 shell vector from your run before publishing.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Understand the login flow

**Do** — The "MediaHub" journalist portal's `login.php` posts FormData `email`/`password` to `api_login.php`, which returns JSON.

```bash
# in Burp, submit any login and watch api_login.php's response:
{"ok":false, ...}
```

**Why / what next** — The front-end decides success from the **`ok`** field in the JSON *response*. That decision happens client-side, so if we tamper the response in transit, the browser proceeds as if login succeeded.

---

## Step 2 — Flag 1 (intercept & flip the response)

**Do** — Turn on Burp's **response interception** for `api_login.php`, submit a login, and edit the response body before it reaches the browser.

```
{"ok":false,...}   ->   {"ok":true, ...}
```

**You get** — The browser treats the login as successful and drops you into the **admin dashboard** → **Flag 1** (`THM{ADMIN_ACCESS_USING_BURP}` format).

**Why / what next** — Authorisation is enforced only in the client's reading of the API response, not server-side on each protected action — so flipping `ok` is enough to get in.

---

## Step 3 — Flag 2 (escalate to a shell)

**Do** — From the admin area, find the feature that reaches the OS / filesystem and drive it to code execution, then read the host flag.

```bash
# target the admin-only feature -> RCE -> reverse shell (nc -lvnp 4444 first)
# flag at /var/www/user.txt
```

**You get** — A shell → **Flag 2** in `/var/www/user.txt` (`THM{SYSTEM_PWNED_SUCCESSFULLY}` format).

> ⚠️ CONFIRM: the exact admin feature + RCE technique used for Q2 on your run.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Read `api_login.php` JSON flow | Login decided client-side by `ok` |
| 2 | Burp response tamper `ok:false→true` | Admin dashboard (Flag 1) |
| 3 | Admin feature → RCE | Shell → `/var/www/user.txt` (Flag 2) |

## Lessons

- When login success is read from a JSON **response field**, Burp **response interception** flips it — auth must be enforced server-side, not in the client.
- Recon that "dead-ends" (SQLi, phpMyAdmin, `.git`) is a signal the intended path is elsewhere — here, traffic interception.

## Remediation

- Enforce authentication/authorisation server-side on every protected request (a session/token the server validates), never from a client-read response flag.
- Lock down admin features that touch the OS/filesystem.

---

*Write-up for [TryHackMe — Interceptor](https://tryhackme.com/room/interceptor).*

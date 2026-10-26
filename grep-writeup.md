# Grep — TryHackMe Writeup

> **Room:** [Grep](https://tryhackme.com/room/grep) · **Difficulty:** Easy–Medium
> **Category:** Web / OSINT & Recon (vhost enum → API key → leak-checker)
> **Author of writeup:** 0xnyx
> **Goal:** Enumerate two vhosts of "SuperSecure Corp", get a working API key to register, and recover the admin's details.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/grep), **no flags or passwords are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon (vhosts from TLS certs)

**Do** — Scan all ports and read the TLS certificate CNs — they name the vhosts.

```bash
nmap -sVC -p- -T4 -Pn <TARGET>
```

**You get**

```
443   https  (cert CN: grep.thm)          -> SearchME app
51337 https  (cert CN: leakchecker.grep.thm) -> password-leak checker
```

**Do** — Add both to hosts.

```bash
printf "<TARGET> grep.thm\n<TARGET> leakchecker.grep.thm\n" | sudo tee -a /etc/hosts
```

**Why / what next** — The cert CN answers the "leak-checker hostname" question directly (`leakchecker.grep.thm`). Now register on the SearchME app.

---

## Step 2 — The right API key

**Do** — The register flow in `register.js` posts to `/api/register.php` with an `X-THM-API-Key` header. The key hardcoded in `register.js` is **rejected** ("Invalid or Expired API key") — it's a decoy.

```bash
curl -s https://grep.thm/register.js | grep -i api
```

**You get** — A decoy key. The **valid** key is exposed elsewhere in the app's client-side assets / API responses (grep every JS file and endpoint, not just `register.js`).

**Why / what next** — With the working key, register and log in:

```bash
curl -sk https://grep.thm/api/register.php \
  -H "X-THM-API-Key: <valid-key>" \
  --data "username=nyx&password=Passw0rd!&email=nyx@grep.thm"
```

**You get** — A valid account → log in → **Flag 1** + the **admin user's email** visible in the app.

---

## Step 3 — Admin password via the leak checker

**Do** — The second vhost, `leakchecker.grep.thm:51337`, has a `check_email.php` that reports breached passwords for an email. Query it for the admin email found in Step 2.

```bash
curl -sk "https://leakchecker.grep.thm:51337/check_email.php" --data "email=<admin-email>"
```

**You get** — The admin's **leaked password** returned by the checker.

**Why / what next** — That's the admin credential — the final answer. (Note `check_email.php` rejects malformed input, so send a well-formed address exactly.)

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | TLS cert CN → vhosts | grep.thm + leakchecker.grep.thm |
| 2 | Find the real API key (decoy in register.js) | Register/login → Flag 1 + admin email |
| 3 | Leak-checker on admin email | Admin password |

## Lessons

- **TLS certificate CNs leak vhosts** — read them on every HTTPS port before fuzzing.
- A key hardcoded in JS may be a **decoy**; grep *all* client assets and API responses for the real one.
- Chain the two apps: the breach checker turns a known email into a password.

## Remediation

- Don't ship API keys (real or decoy) in client-side JS; authenticate server-side.
- A public "leak checker" must not return actual plaintext passwords for arbitrary emails.
- Don't reuse breached passwords for admin accounts.

---

*Write-up for [TryHackMe — Grep](https://tryhackme.com/room/grep).*

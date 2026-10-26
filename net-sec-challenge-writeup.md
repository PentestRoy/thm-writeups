# Net Sec Challenge — TryHackMe Writeup

> **Room:** [Net Sec Challenge](https://tryhackme.com/room/netsecchallenge) · **Difficulty:** Easy
> **Category:** Network Security skills-check (nmap · telnet · hydra)
> **Author of writeup:** 0xnyx
> **Goal:** Answer the challenge questions using the Network Security module's tools — port scanning, banner grabbing, and a login brute-force.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/netsecchallenge), **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. This is a skills-check, so each step maps to a question.

---

## Step 1 — Enumerate every open port

**Do** — Full TCP scan with service/version detection.

```bash
nmap -p- -sVC -T4 -Pn <TARGET>
```

**You get** — The complete list of open ports (count + the highest-numbered one), plus service versions. This answers the "how many ports / highest port / service version" questions.

**Why / what next** — Note any service whose **banner** might carry a flag (SSH, FTP, a custom high port), and any login service to brute.

---

## Step 2 — Grab flags from service banners

**Do** — Connect to the service and read what it prints. SSH banners and custom ports often hold a flag directly.

```bash
# SSH version/banner:
nmap -p22 --script banner <TARGET>
# raw banner grab on any port:
telnet <TARGET> <PORT>
nc -nv <TARGET> <PORT>
```

**You get** — The flag embedded in the banner / first response.

> ⚠️ CONFIRM: fill in which port carried the banner flag on your run, and which question it answered.

---

## Step 3 — Brute-force a login

**Do** — Use hydra against the login service the challenge points at (e.g. SSH/FTP) with the given/rockyou wordlist.

```bash
hydra -l <user> -P /usr/share/wordlists/rockyou.txt <service>://<TARGET>
```

**You get** — Valid credentials → log in → read the final flag.

> ⚠️ CONFIRM: fill in the exact service/user and where the final flag lived on your run.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | `nmap -p- -sVC` | Port count, highest port, versions |
| 2 | telnet/nc/`--script banner` | Banner flag(s) |
| 3 | hydra login brute | Creds → final flag |

## Lessons

- `-p-` matters — challenge flags often hide on **high, non-default ports** a top-1000 scan misses.
- Service **banners** are free flags; grab them with telnet/nc or `--script banner`.
- hydra + rockyou is the standard credential brute for this module.

## Remediation

- Don't expose internal services on odd ports; don't put secrets in banners.
- Rate-limit logins (fail2ban) to defeat brute-force.

---

*Write-up for [TryHackMe — Net Sec Challenge](https://tryhackme.com/room/netsecchallenge).*

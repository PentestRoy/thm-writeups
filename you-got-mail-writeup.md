# You Got Mail — TryHackMe Writeup

> **Room:** [You Got Mail](https://tryhackme.com/room/yougotmail) · **Difficulty:** Medium
> **Category:** Windows / Mail (hMailServer enumeration → credential reuse)
> **Author of writeup:** 0xnyx
> **Goal:** Enumerate a Windows hMailServer stack, recover mail credentials, and reuse them to land a shell.

> ⚠️ In line with TryHackMe's [write-up policy](https://tryhackme.com/room/yougotmail), **no flags or passwords are included** — only the method.
>
> ⚠️ CONFIRM: this is a skeleton from the general hMailServer path — fill in the exact creds source and shell vector from your run before publishing.

---

## How to read this writeup

Each step: **Do** → **You get** → **Why / what next**.

---

## Step 1 — Recon

**Do** — Full scan; a Windows mail box exposes SMB + the mail trio.

```bash
nmap -p- -sVC -T4 -Pn <TARGET>
```

**You get** — Expect SMB (139/445), SMTP (25/587), POP3 (110/995), IMAP (143/993), and likely RDP (3389) / WinRM (5985) for the eventual shell.

**Why / what next** — Enumerate SMB and the mail services for usernames and a first credential.

> ⚠️ CONFIRM: list the actual open ports from your scan.

---

## Step 2 — Recover mail credentials

**Do** — Enumerate shares / mailboxes and read any leaked creds.

```bash
smbclient -L //<TARGET>/ -N
# read mailboxes over POP3/IMAP once you have one credential:
# telnet <TARGET> 110   -> USER/PASS -> RETR
```

**You get** — A mailbox credential (from a share, a note, or a weak account).

> ⚠️ CONFIRM: where the first credential came from (SMB file / brute / email body).

**Why / what next** — Read the inbox; mail boxes often chain one account's message into another account's password.

---

## Step 3 — Foothold

**Do** — Reuse the recovered credentials against the remote-access service.

```bash
# WinRM:
evil-winrm -i <TARGET> -u <user> -p <pass>
# or RDP:
xfreerdp /u:<user> /p:<pass> /v:<TARGET>
```

**You get** — A shell on the host → user flag; privesc as needed for the root/Administrator flag.

> ⚠️ CONFIRM: the exact shell vector and privesc step from your run.

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | `nmap -p-` | SMB + mail services + remote-access port |
| 2 | SMB / mail enumeration | Mailbox credential(s) |
| 3 | Credential reuse (WinRM/RDP) | Shell → flags |

## Lessons

- A Windows **hMailServer** box is a credential-chaining exercise — mailbox → mailbox → account.
- Always try recovered creds against WinRM/RDP; password reuse is the norm.

## Remediation

- Enforce unique, strong mailbox passwords; don't store creds in shares or email bodies.
- Restrict RDP/WinRM exposure and enforce MFA.

---

*Write-up for [TryHackMe — You Got Mail](https://tryhackme.com/room/yougotmail).*

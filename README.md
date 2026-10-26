<h1 align="center">TryHackMe Write-ups</h1>

<p align="center">
  Detailed, reproducible, policy-safe walkthroughs of the TryHackMe rooms I've solved.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/platform-TryHackMe-212c42?style=flat-square&logo=tryhackme" alt="TryHackMe">
  <img src="https://img.shields.io/badge/write--ups-33-blue?style=flat-square" alt="Write-ups">
  <img src="https://img.shields.io/badge/focus-Web%20%7C%20AD%20%7C%20AI-success?style=flat-square" alt="Focus">
  <img src="https://img.shields.io/badge/policy-flag--safe-green?style=flat-square" alt="Policy safe">
</p>

---

## About

I'm **0xnyx** ([@PentestRoy](https://github.com/PentestRoy)) — I focus on web application security and am training as an ethical hacker (Red Team).

This repository is my working log: for every room I finish, I write up the **full attack chain** the way I'd present it in an engagement report — enumeration, the decision behind each step, the exact technique, and how I'd remediate it. The goal is that anyone can follow the method and understand *why* each move was made, not just copy a command.

> **Responsible disclosure:** In line with [TryHackMe's write-up policy](https://tryhackme.com/r/resources/blog), these write-ups contain **no flags, passwords, or cracked values** — only the methodology against the authorised lab environment.

---

## How each write-up is structured

Every write-up follows the same, deliberately repeatable shape:

- **Do → You get → Why / what next** — each step shows the command, the real output to expect, and the reasoning that drives the next move.
- **Full-chain table** — the whole path from anonymous to objective, at a glance.
- **Lessons** — the transferable techniques the box teaches.
- **Remediation** — how a defender would close each hole.

---

## Full index

| Box | Difficulty | Focus | Write-up |
|-----|------------|-------|----------|
| Room 404 | `Very Easy` | Web / Directory Enumeration (exposed `.git`) | [→](room-404-writeup.md) |
| Pickle Rick | `Easy` | Web / Command Injection | [→](pickle-rick-writeup.md) |
| Skynet | `Easy` | Web / Linux (SMB → Cuppa RFI → tar wildcard) | [→](skynet-writeup.md) |
| Basic Pentesting | `Easy` | Linux / Enumeration (SMB → brute → SSH key crack) | [→](basic-pentesting-writeup.md) |
| Vulnerability Capstone | `Easy` | Web (FUEL CMS 1.4 CVE-2018-16763 RCE) | [→](vulnerability-capstone-writeup.md) |
| RootMe | `Easy` | Web (upload bypass → SUID privesc) | [→](rootme-writeup.md) |
| Net Sec Challenge | `Easy` | Network Security skills-check (nmap · telnet · hydra) | [→](net-sec-challenge-writeup.md) |
| Evil-GPT v2 | `Easy` | AI / LLM Security (guardrail bypass) | [→](evil-gpt-v2-writeup.md) |
| Creative | `Easy–Medium` | Web / Boot2Root (SSRF → LD_PRELOAD) | [→](creative-writeup.md) |
| CyberLens | `Easy–Medium` | Web / Windows (Apache Tika RCE → AlwaysInstallElevated) | [→](cyberlens-writeup.md) |
| Grep | `Easy–Medium` | Web / OSINT & Recon (vhost enum → API key → leak-checker) | [→](grep-writeup.md) |
| Do Not Disturb | `Medium` | Web / Boot2Root (Node.js) | [→](do-not-disturb-writeup.md) |
| Domino | `Medium` | Web / Chained exploitation | [→](domino-writeup.md) |
| Frankesqwen | `Medium` | AI / LLM Security (prompt jailbreak) | [→](frankesqwen-writeup.md) |
| Injectics | `Medium` | Web / Injection (SQLi → Twig SSTI) | [→](injectics-writeup.md) |
| Ledger | `Medium` | Active Directory (ADCS ESC1 → pass-the-cert) | [→](ledger-writeup.md) |
| Oh My WebServer | `Medium` | Web / Container Escape (CVE-2021-42013 → OMIGOD) | [→](oh-my-webserver-writeup.md) |
| Silent Monitor | `Medium` | Web / Boot2Root (Flask) | [→](silent-monitor-writeup.md) |
| Whats Your Name? | `Medium` | Web / Client-side (XSS + CSRF) | [→](whats-your-name-writeup.md) |
| Mr Robot CTF | `Medium` | Web / WordPress (brute → theme shell → SUID nmap) | [→](mr-robot-writeup.md) |
| Hammer | `Medium` | Web / Auth bypass → JWT forgery → RCE | [→](hammer-writeup.md) |
| Olympus | `Medium` | Web / Boot2Root (CMS SQLi → SUID → backdoor) | [→](olympus-writeup.md) |
| Recruit | `Medium` | Web (log leak → LFI → SQLi) | [→](recruit-writeup.md) |
| Burg3r Bytes | `Medium` | Web / Business-logic (race → SSTI → protocol traversal) | [→](burg3r-bytes-writeup.md) |
| You Got Mail | `Medium` | Windows / Mail (hMailServer → credential reuse) | [→](you-got-mail-writeup.md) |
| Interceptor | `Medium` | Web / Burp (response tampering → RCE) | [→](interceptor-writeup.md) |
| Jurassic Park | `Medium–Hard` | Web / SQL Injection | [→](jurassic-park-writeup.md) |
| Contrabando | `Hard` | Web / Boot2Root (smuggling → SSRF/SSTI → sudo) | [→](contrabando-writeup.md) |
| El Bandito | `Hard` | Web / Red Teaming (request smuggling) | [→](el-bandito-writeup.md) |
| Farewell | `Hard (MAX)` | Web / Red Teaming (WAF bypass) | [→](farewell-writeup.md) |
| Extract | `MAX (Hard)` | Web (SSRF → gopher → Next.js bypass → cookie tampering) | [→](extract-writeup.md) |
| Sequence | `MAX (Hard)` | Web / Boot2Root (XSS → CSRF → SSRF → Docker escape) | [→](sequence-writeup.md) |
| Voyage | `MAX (Hard)` | Web / Boot2Root (Joomla → container → kernel-module escape) | [→](voyage-writeup.md) |

---

## By focus

| Area | Rooms |
|------|-------|
| 🌐 Web / Injection & SQLi | Pickle Rick · Jurassic Park · Injectics · Olympus · Recruit |
| 🔗 Web / Boot2Root & Chained | Skynet · Creative · Do Not Disturb · Domino · Silent Monitor · Oh My WebServer · Mr Robot · Contrabando · Sequence · Voyage · Burg3r Bytes |
| 🔐 Web / Auth & Red Team | Hammer · Whats Your Name? · El Bandito · Farewell · Extract · Interceptor |
| 🪟 Windows | CyberLens · You Got Mail |
| 🏢 Active Directory | Ledger |
| 🤖 AI / LLM Security | Frankesqwen · Evil-GPT v2 |
| 🧰 Fundamentals / Recon | Basic Pentesting · RootMe · Vulnerability Capstone · Room 404 · Net Sec Challenge · Grep |

---

## Disclaimer

All techniques shown were performed against the **authorised TryHackMe lab environment**, on machines I had explicit permission to attack. These write-ups are for education and defensive awareness only. Do not use them against systems you do not own or have written authorisation to test.

---

<p align="center"><sub>Maintained by 0xnyx · built one box at a time.</sub></p>

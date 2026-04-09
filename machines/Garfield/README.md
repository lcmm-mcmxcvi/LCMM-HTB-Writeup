# HTB: Garfield

| Info | Detail |
|------|--------|
| OS | Windows Server 2019 (Build 17763) |
| Difficulty | TBD |
| Domain | garfield.htb |
| DC | DC01.garfield.htb |
| Date | 2026-04-08 |
| Status | 🔄 In Progress |
| Initial Creds | j.arbuckle |

## Attack Chain (Planned)

```
j.arbuckle → WriteProperty scriptPath → l.wilson logon script execution
→ l.wilson shell (WinRM) → ? → l.wilson_adm (Tier 1)
→ RODC pivot (192.168.100.2) → Cached Administrator hash → Domain Admin
```

---

## Reconnaissance

### Port Scan

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: garfield.htb)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2179/tcp  open  vmrdp?        (Hyper-V VM Remote Desktop)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
9389/tcp  open  mc-nmf        .NET Message Framing
```

### Domain Users

| User | UAC | Groups | Access |
|------|-----|--------|--------|
| Administrator | 66048 | Domain Admins, Enterprise Admins, Schema Admins | Full |
| j.arbuckle (us) | 66048 | IT Support | SMB ✅, WinRM ❌, RDP ❌ |
| l.wilson | 66048 | Remote Desktop Users, Remote Management Users | WinRM ✅, RDP ✅ |
| l.wilson_adm | 66048 | Tier 1, Remote Desktop Users, Remote Management Users | WinRM ✅, RDP ✅ |
| krbtgt_8245 | 66050 | (none) | RODC KDC account |

### Domain Computers

| Computer | Role | Network |
|----------|------|---------|
| DC01 | Domain Controller | External (10.129.x.x) |
| RODC01 | Read-Only Domain Controller | Internal (192.168.100.2) |

### RODC Analysis

**Cached passwords (msDS-RevealedUsers):**
- `krbtgt_8245` — RODC KDC account
- `RODC01$` — RODC machine account
- `Administrator` — **Domain Admin password cached on RODC**

**Replication policy:**
- Allowed RODC Password Replication Group: **empty**
- No `managedBy` set

### ACL Findings

**IT Support** (j.arbuckle's group):
- **WriteProperty on Script-Path** for all User objects (inherited from domain)
- No write to: SPN, password, shadow credentials, RODC replication groups

### Checks Performed

| Check | Result |
|-------|--------|
| ADCS | Not present |
| LAPS | Not configured |
| GPP Passwords | None |
| AS-REP Roastable | None |
| Kerberoastable | None |
| Constrained Delegation | None |
| RBCD | None configured |
| Account Lockout | None (unlimited) |
| Targeted Kerberoast (set SPN) | `insufficientAccessRights` |
| Password change (LDAP/RPC) | `insufficientAccessRights` |
| Shadow Credentials (pywhisker) | `insufficientAccessRights` |
| Add to RODC Administrators | `insufficientAccessRights` |

---

## Progress

### NTLMv2 Hash Captured

Login simulation triggered l.wilson's logon script → SMB auth to our impacket smbserver → NTLMv2 hash captured:

```
l.wilson::GARFIELD:aaaaaaaaaaaaaaaa:f4d5e3cbbb78c2579a6e5f26701b46a0:0101000000000000804dfaca07c8dc01...
```

**Cracking attempts (all exhausted):**
| Wordlist/Rules | Keyspace | Result |
|----------------|----------|--------|
| rockyou.txt (14M) | 14M | ❌ |
| rockyou + best64 rules | ~920M | ❌ |
| rockyou + d3ad0ne rules | ~490B | ❌ |
| Themed Garfield wordlist | ~60 | ❌ |
| Themed + best64 rules | ~3.8K | ❌ |

### Reverse Shell Connection

PowerShell reverse shell connected (TCP handshake confirmed on port 8081) but no data stream — likely encoding issue with raw nc handling PowerShell UTF-16 output.

### Key Observations

- Login simulation fires **once per machine reset** — payload must be ready before setting scriptPath
- SYSVOL `garfield.htb\scripts\` is writable by j.arbuckle
- HTTP outbound from target works (confirmed shell.ps1 download on port 8080)
- Arbitrary TCP outbound may be filtered (port 9001 failed, 8081 connected but no data)

---

## Next Steps (Resume Plan)

1. **Reset machine** — simulation only fires once
2. **Pre-stage payload** — upload SYSVOL file-exfil bat BEFORE setting scriptPath:
   ```bat
   @echo off
   powershell -ep bypass -nop -c "Get-Content C:\Users\l.wilson\Desktop\user.txt | Out-File -Encoding ascii C:\Windows\SYSVOL\sysvol\garfield.htb\scripts\user.txt"
   ```
3. **Set scriptPath** → simulation fires → user.txt written to SYSVOL → read via SMB
4. **Alternative:** Fix impacket 0.12.0 for ntlmrelayx SMB→WinRM relay
5. **After l.wilson shell:** enumerate path to l.wilson_adm → RODC pivot → cached Admin hash

## Infrastructure Fixes Needed

- Fix rockyou.txt symlink (broken → /tmp/rockyou.txt)
- Redownload kaonashi wordlist (corrupted)
- Upgrade impacket ntlmrelayx to 0.12.0 (0.11.0 won't bind 445)

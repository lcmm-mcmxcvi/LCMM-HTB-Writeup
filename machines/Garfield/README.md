# HTB: Garfield

| Info | Detail |
|------|--------|
| OS | Windows Server 2019 |
| Difficulty | TBD |
| Domain | garfield.htb |
| DC | DC01.garfield.htb |
| Date | 2026-04-08 |
| Status | 🔄 In Progress |
| Initial Creds | j.arbuckle |

## Enumeration

### Domain Info
- **DC01** — Windows Server 2019 Build 17763, Domain Controller
- **RODC01** — Read-Only Domain Controller at 192.168.100.2 (internal network)
- Clock skew: +8 hours
- SMB signing: required
- No ADCS, no LAPS, no GPP passwords

### Users
| User | Groups | Access |
|------|--------|--------|
| j.arbuckle (initial) | IT Support | SMB ✅, WinRM ❌ |
| l.wilson | Remote Desktop Users, Remote Management Users | WinRM ✅, RDP ✅ |
| l.wilson_adm | Tier 1, Remote Desktop Users, Remote Management Users | WinRM ✅, RDP ✅ |
| krbtgt_8245 | — | RODC KDC account |

### RODC Analysis
- RODC01 at 192.168.100.2 (not directly reachable)
- Cached passwords (msDS-RevealedUsers): **krbtgt_8245**, **RODC01$**, **Administrator**
- Allowed RODC Password Replication Group: empty
- No managedBy set

### ACL Findings
- **IT Support** has **WriteProperty on Script-Path** for all User objects (inherited from domain)
- No write access to SPN, password, shadow credentials, or RODC replication groups

## Attack Path (In Progress)

```
j.arbuckle → WriteProperty scriptPath → l.wilson logon script → l.wilson shell → ? → l.wilson_adm → RODC pivot → Administrator hash from cache → Domain Admin
```

### Current Status
- Logon script attack set up (SYSVOL write + scriptPath modification)
- Awaiting login simulation trigger
- Alternative vectors explored: targeted Kerberoast (denied), password change (denied), shadow creds (denied)

## TODO
- Get l.wilson shell via logon script
- Pivot to l.wilson_adm (Tier 1)
- Reach RODC01 on internal network
- Extract cached Administrator hash from RODC

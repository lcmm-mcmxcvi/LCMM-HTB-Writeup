# HTB: Pirate

| Info | Detail |
|------|--------|
| OS | Windows |
| Difficulty | Hard |
| Domain | pirate.htb |
| Date | 2026-03-02 |
| Type | Pure Active Directory |

## Attack Chain

```
Pre-2K Compat → Machine Account (MS01$) → gMSA Password Read → Ligolo Pivot → NTLM Relay (WEB01→DC LDAP) → Shadow Credentials + RBCD → a.white creds → ForceChangePassword a.white_adm → WriteSPN on DC01$ → KCD + SPN Hijack → Domain Admin
```

## Phase 1 — Pre-Windows 2000 Compatibility

- Identified "Pre-Windows 2000 Compatible Access" group membership
- Found pre-created computer accounts: `MS01$` and `ES01$`
- Default password = sAMAccountName minus `$` (i.e., password `ms01` for `MS01$`)
- Authenticated as machine account using NetExec pre2k module

## Phase 2 — gMSA Password Retrieval

- `MS01$` had `msDS-AllowedToRetrieveManagedPassword` rights
- Retrieved NTLM hash for `gMSA_ADFS_prod$`
- Used gMSA credentials to access the DC

## Phase 3 — Internal Network Pivot

- Discovered dual-homed network (192.168.x.x)
- Deployed **Ligolo-ng v0.6.2** for pivoting (v0.7.5 crashes — permanent note)
- Found `WEB01` on internal network with **SMB signing disabled**

## Phase 4 — NTLM Relay

- Coerced `WEB01` to authenticate using PetitPotam/PrinterBug
- Relayed NTLM authentication to DC's LDAP (`--remove-mic` flag)
- Obtained LDAP shell as `WEB01$`
- Used **Impacket v0.12.0** (v0.11.0 has ntlmrelayx bugs)

## Phase 5 — Shadow Credentials + RBCD

- Added KeyCredential to `WEB01`'s `msDS-KeyCredentialLink`
- Configured Resource-Based Constrained Delegation on `WEB01` to trust itself
- S4U2Self/S4U2Proxy to impersonate Domain Admin on `WEB01`

## Phase 6 — Credential Harvesting

- Dumped secrets on `WEB01`
- Obtained cleartext password for `a.white`
- `a.white` had **ForceChangePassword** on `a.white_adm`

## Phase 7 — Domain Admin via KCD + SPN Hijacking

- Reset `a.white_adm` password
- `a.white_adm` had **WriteSPN** on `DC01$`
- Moved `HTTP/WEB01` SPN from `WEB01$` to `DC01$`
- Used Protocol Transition to obtain ticket for Administrator
- Rewrote service to `CIFS/DC01` with `-altservice`
- PSExec to DC as SYSTEM

## Key Corrections

- Use `nxc -X` (not evil-winrm) for gMSA WinRM sessions
- Impacket v0.12.0 via pip (v0.11.0 ntlmrelayx is buggy)
- Ligolo v0.6.2 (v0.7.5 crashes)
- `faketime` required to bypass persistent Kerberos clock skew

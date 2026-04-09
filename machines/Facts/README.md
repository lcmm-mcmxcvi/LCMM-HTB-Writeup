# HTB: Facts

| Info | Detail |
|------|--------|
| OS | Linux |
| Difficulty | Medium |
| IP | 10.129.x.x |
| Date | 2026-03-09 |
| Stack | Rails / Camaleon CMS / MinIO |

## Attack Chain

```
LFI → SQLite DB → MinIO Admin Creds → SSH Key Extraction → Bcrypt Passphrase Crack → SSH User → Root
```

## Enumeration

- Web application running **Ruby on Rails** with **Camaleon CMS**
- **MinIO** object storage backend discovered

## Foothold — LFI to MinIO

- Identified Local File Inclusion vulnerability in the Rails application
- Extracted SQLite database via LFI
- Found MinIO admin credentials in the database
- Accessed MinIO admin panel and extracted SSH private key

## User — SSH Key Cracking

- SSH key was encrypted with a bcrypt passphrase
- **hashcat does NOT support bcrypt-encrypted OpenSSH keys** (mode 22921 is for non-bcrypt only)
- Used `john-the-ripper` (jumbo) as primary cracker
- Built parallel Python cracker (`ssh-crack.py`) using all CPU cores (~80/s on i9-14900K)
- Cracked passphrase in 62 seconds
- SSH access as user

## Root

- Privilege escalation completed (details in attack chain)

## Lessons Learned

### Cracking Decision Tree (Permanent)
1. Check if hashcat mode exists for the hash type
2. If not → try john-jumbo (`/snap/bin/john-the-ripper`)
3. If neither → use parallel Python fallback
4. **Quick win**: try top 1000-5000 passwords first — most HTB passwords are in rockyou top 5000

### Tool Issues Encountered
- `ssh2john` path: `/snap/john-the-ripper/current/bin/ssh2john.py`
- VPN dropped silently during SSH attempts → created `vpn-check.sh` auto-reconnect
- SSH agent died between execute_bash calls → created `ssh-persist.sh` with env file persistence
- Long commands with no output caused user interrupts → created `progress-wrap.sh`

### Permanent Rules Added
- Never crack via Docker — always native for GPU + all CPU cores
- VPN health check before every remote command
- All SSH via agent with persisted env file
- All long commands wrapped with progress output

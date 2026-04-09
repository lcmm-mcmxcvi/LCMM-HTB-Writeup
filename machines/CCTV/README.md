# HTB: CCTV

| Info | Detail |
|------|--------|
| OS | Linux |
| Difficulty | Medium |
| IP | 10.129.4.2 |
| Date | 2026-03-09 |
| User Flag | `778dce68101fcde804d0f862c5d04b79` |
| Root Flag | `4c9f9ea5c6eb20ee80504c1259af942d` |

## Attack Chain

```
ZoneMinder SQLi → Hash Crack → SSH (mark) → motionEye Internal (port 8765) → HMAC Signature Forge → command_notifications RCE → Root
```

## Enumeration

- Web application running **ZoneMinder** (CCTV management software)
- Standard port scan revealed HTTP service

## Foothold — ZoneMinder SQL Injection

- Identified SQL injection vulnerability in ZoneMinder
- Extracted password hashes from the database
- Cracked hash to obtain credentials for user `mark`
- SSH access as `mark` using cracked credentials

## Privilege Escalation — motionEye RCE

### Internal Service Discovery

After getting a shell as `mark`, ran `ss -tlnp` and discovered **motionEye** running on internal port **8765** — not accessible externally.

### HMAC Signature Forgery

- Read the motionEye application source code to understand the authentication mechanism
- Discovered the API used HMAC-based signature verification for command execution
- Forged valid HMAC signatures to authenticate to the internal API

### Root via command_notifications

- Exploited the `command_notifications` feature in motionEye
- Injected commands through the notification handler
- Achieved RCE as `root`

## Reusable Scripts Created

- `post-shell-enum.sh` — Automated post-exploitation enumeration
- `motioneye-rce.py` — motionEye command_notifications exploit

## Lessons Learned

- `ss -tlnp` + `ps aux` should be the FIRST commands after any shell — internal services are the most common Linux privesc vector
- When facing custom auth on internal services, read the application source code immediately — don't guess authentication schemes
- Never embed `sleep` inside SSH sessions — split into separate calls

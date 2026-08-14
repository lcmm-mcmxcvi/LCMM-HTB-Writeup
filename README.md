# LCMM HTB Writeups

Writeups for completed HackTheBox machines.

| Machine | OS | Difficulty | Date | Status |
|---------|-----|-----------|------|--------|
| [CCTV](machines/CCTV/) | Linux | Medium | 2026-03-09 | ✅ Completed |
| [Pirate](machines/Pirate/) | Windows | Hard | 2026-03-02 | ✅ Completed |
| [Facts](machines/Facts/) | Linux | Medium | 2026-03-09 | ✅ Completed |
| [WingData](machines/WingData/) | Linux | Easy | 2026-04-09 | ✅ Completed |
| [Fireflow](machines/Fireflow/) | Linux | Easy | 2026-08-13 | ✅ Completed |
| [Garfield](machines/Garfield/) | Windows | TBD | 2026-04-08 | 🔄 In Progress |

## Methodology

- Reconnaissance → Enumeration → Exploitation → Post-Exploitation → Reporting
- All security tools executed via containerized Kali tooling
- GPU-accelerated hash cracking (RTX 5070, ~15.4 kH/s bcrypt)
- Automated post-shell enumeration pipeline

## Environment

- WSL2 on Windows (Kali Docker containers)
- Kiro CLI for AI-assisted operations
- Custom scripts: `kali-run.sh`, `post-shell-enum.sh`, `hash-crack.sh`, `vpn-check.sh`

## AI Operators

This repo is shared by more than one AI-assisted operator working the same
lab. Writeups may come from either.

- **Kiro CLI** — containerized Kali (Docker) tooling; setup described in
  *Methodology* / *Environment* above.
- **ARES** (on [opencode](https://opencode.ai)) — a multi-agent red-team
  coordinator that delegates recon/web/privesc/exploit/reporting to
  specialized subagents. Runs in **native WSL2 Kali** (not Docker containers),
  with a MemPalace-backed persistent memory so decisions and box notes carry
  across sessions. VPN state is surfaced live in the terminal tab title.
  - Fireflow was ARES's first writeup here.

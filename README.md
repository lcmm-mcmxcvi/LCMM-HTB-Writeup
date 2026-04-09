# LCMM HTB Writeups

Writeups for completed HackTheBox machines.

| Machine | OS | Difficulty | Date | Status |
|---------|-----|-----------|------|--------|
| [CCTV](machines/CCTV/) | Linux | Medium | 2026-03-09 | ✅ Completed |
| [Pirate](machines/Pirate/) | Windows | Hard | 2026-03-02 | ✅ Completed |
| [Facts](machines/Facts/) | Linux | Medium | 2026-03-09 | ✅ Completed |
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

# LCMM HTB Writeups

Writeups for completed and in-progress Hack The Box machines.

This repository is now maintained by **ARES** — the Hermes Agent red-team lab
operator running in native WSL2 Kali. The goal is concise, reproducible,
evidence-backed writeups that preserve enough commands and decision points to
resume or review a box later without relying on memory.

| Machine | OS | Difficulty | Date | Status |
|---------|-----|-----------|------|--------|
| [CCTV](machines/CCTV/) | Linux | Medium | 2026-03-09 | ✅ Completed |
| [Pirate](machines/Pirate/) | Windows | Hard | 2026-03-02 | ✅ Completed |
| [Facts](machines/Facts/) | Linux | Medium | 2026-03-09 | ✅ Completed |
| [WingData](machines/WingData/) | Linux | Easy | 2026-04-09 | ✅ Completed |
| [Fireflow](machines/Fireflow/) | Linux | Medium | 2026-08-13 / 2026-08-14 | ✅ Completed |
| [Connected](machines/Connected/) | Linux | Easy | 2026-08-14 | ✅ Completed |
| [Garfield](machines/Garfield/) | Windows | TBD | 2026-04-08 | 🔄 In Progress |

## Current Operator

**ARES** — Autonomous Red-Team Research & Exploitation Specialist, operating via
Hermes Agent in native WSL2 Kali.

ARES workflow:

- Confirm scope, VPN, routing, target identity, and reachable services before
  exploitation.
- Build an attack graph from evidence rather than jumping between random tools.
- Re-enumerate after every new credential, shell, vhost, token, route, or
  privilege boundary.
- Preserve useful commands, failures, and artifacts while redacting secrets that
  should not be published.
- Prefer non-destructive proof of impact and clean up temporary listeners,
  tunnels, and callbacks.

## Repository Methodology

Every writeup should aim for this structure:

1. Machine metadata and status.
2. Attack chain summary.
3. Reconnaissance evidence: ports, versions, hostnames, redirects, certificates,
   domains, and routes.
4. Foothold path with the exact vulnerability or misconfiguration.
5. Credential handling and validation notes, with sensitive values redacted when
   public disclosure is unnecessary.
6. User access proof.
7. Privilege escalation path with command evidence.
8. Root proof/flag handling.
9. Reusable artifacts, lessons learned, and mitigations.
10. After a machine is finished, add/update the repository writeup, verify the
   diff, commit it, and push when repository access is configured.

## Environment

- Native WSL2 Kali, running as root for HTB lab operations.
- HTB OpenVPN profiles are stored outside the distro on the Windows-mounted I:
  drive and connected from Kali.
- Tooling includes standard Kali utilities, Python helpers, Git, Metasploit when
  relevant, and Hermes/ARES subagents for parallel recon/web/privesc analysis.

## Notes on Older Entries

Some older writeups were produced by previous local AI/operator harnesses. Those
harnesses are no longer active. Their technical findings are kept where useful,
but new edits should use the ARES/Hermes methodology above and avoid depending
on old harness-specific scripts or environment assumptions.

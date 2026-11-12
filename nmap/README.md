# Nmap

Notes on **Nmap (Network Mapper)**, the open-source tool for host discovery, port scanning, service/version detection, OS fingerprinting, and scripting.

## Contents

| File | Description |
|---|---|
| [nmap-notes.md](./nmap-notes.md) | Full notes: scan types, port states, NSE scripts, timing, output formats, evasion concepts, recipes, cheat sheet |

## What's covered

- Installing Nmap and basic syntax
- Target and port specification
- Host discovery (`-sn`, `-Pn`, `-PR`...)
- Scan types: SYN, Connect, UDP, Null/FIN/Xmas, ACK
- Service/version and OS detection
- Timing templates and performance tuning
- Nmap Scripting Engine (NSE)
- Output formats (`-oN`, `-oX`, `-oG`, `-oA`)
- Firewall/IDS evasion concepts (from a defender's view too)
- Ready-to-use command recipes and a cheat sheet

## Quick start

```bash
# Discover live hosts
sudo nmap -sn 192.168.1.0/24

# Standard recon scan
sudo nmap -sC -sV -O -oA recon <target>
```

## Legal & ethical use

Only scan systems you own or have **explicit written permission** to test. Use safe targets such as `scanme.nmap.org`, local lab VMs, or CTF platforms.

## Resources

- https://nmap.org
- https://nmap.org/book/man.html
- https://nmap.org/nsedoc/

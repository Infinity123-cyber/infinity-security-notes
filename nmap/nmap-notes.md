# Nmap Notes

> **Legal note:** Only scan systems you own or have **written permission** to test. Unauthorized scanning can be illegal. For practice, use `scanme.nmap.org` (allowed by the Nmap project, light scans only), your own lab VMs, or platforms like TryHackMe / Hack The Box.

---

## 1. What is Nmap?

**Nmap (Network Mapper)** is a free, open-source tool for:

- **Host discovery**: which devices are alive on a network
- **Port scanning**: which ports are open, closed, or filtered
- **Service/version detection**: what software is running on a port
- **OS detection**: guessing the operating system
- **Scripting (NSE)**: automating checks, vulnerability detection, enumeration

Used by network admins, SOC analysts, pentesters, and bug hunters.

### Install

```bash
# Debian / Ubuntu / Kali
sudo apt update && sudo apt install nmap

# macOS
brew install nmap

# Windows: download installer from https://nmap.org/download.html

# Check version
nmap --version
```

---

## 2. Basic Syntax

```bash
nmap [scan type] [options] <target>
```

Many scans (SYN scan, OS detection) need **root/administrator** privileges, so use `sudo`.

---

## 3. Specifying Targets

| Format | Example | Meaning |
|---|---|---|
| Single IP | `nmap 192.168.1.10` | One host |
| Hostname | `nmap scanme.nmap.org` | One host by name |
| Range | `nmap 192.168.1.1-50` | IPs .1 to .50 |
| CIDR | `nmap 192.168.1.0/24` | Whole /24 subnet (256 addresses) |
| Multiple | `nmap 10.0.0.1 10.0.0.5` | Listed hosts |
| From file | `nmap -iL targets.txt` | Targets listed in a file |
| Exclude | `nmap 192.168.1.0/24 --exclude 192.168.1.1` | Skip a host |
| Random | `nmap -iR 10` | 10 random hosts (be careful!) |

---

## 3.5 Port States

| State | Meaning |
|---|---|
| **open** | A service is actively accepting connections |
| **closed** | Reachable, but no service listening |
| **filtered** | Firewall/filter is blocking, so Nmap can't tell |
| **unfiltered** | Reachable, but Nmap can't tell if open or closed (ACK scan) |
| **open\|filtered** | Can't decide between open and filtered |
| **closed\|filtered** | Can't decide between closed and filtered |

---

## 4. Host Discovery (Ping Scan)

Find live hosts **without** port scanning:

```bash
nmap -sn 192.168.1.0/24
```

| Flag | Purpose |
|---|---|
| `-sn` | Ping scan only (no port scan) |
| `-Pn` | Skip host discovery, treat all hosts as up (useful when ICMP is blocked) |
| `-PE` | ICMP echo request |
| `-PP` | ICMP timestamp request |
| `-PS<ports>` | TCP SYN ping, e.g. `-PS22,80,443` |
| `-PA<ports>` | TCP ACK ping |
| `-PU<ports>` | UDP ping |
| `-PR` | ARP ping (default on local network, very reliable) |
| `-n` | Never do DNS resolution (faster) |
| `-R` | Always do DNS resolution |

> On a local LAN, Nmap uses **ARP** for discovery. On remote networks it uses ICMP echo + TCP SYN to 443 + TCP ACK to 80 + ICMP timestamp by default.

---

## 5. Scan Types

### 5.1 TCP SYN Scan (`-sS`): "Stealth" / half-open scan

```bash
sudo nmap -sS 192.168.1.10
```

- Sends SYN, waits for reply, never completes the handshake.
- **SYN/ACK** → open. **RST** → closed. **No reply / ICMP unreachable** → filtered.
- Default scan when run as root. Fast and fairly quiet (but still detectable by IDS).

### 5.2 TCP Connect Scan (`-sT`)

```bash
nmap -sT 192.168.1.10
```

- Completes the full 3-way handshake using the OS's `connect()` call.
- Default when **not** root. Slower and more likely to be logged by the target.

### 5.3 UDP Scan (`-sU`)

```bash
sudo nmap -sU 192.168.1.10
sudo nmap -sU --top-ports 20 192.168.1.10
```

- Slow because UDP has no handshake and closed ports rate-limit ICMP replies.
- Important services: DNS (53), DHCP (67/68), SNMP (161), TFTP (69), NTP (123).
- Combine with TCP: `sudo nmap -sS -sU -p T:80,443,U:53,161 <target>`

### 5.4 Other TCP Scans

| Flag | Name | Idea |
|---|---|---|
| `-sN` | Null scan | No flags set |
| `-sF` | FIN scan | Only FIN flag |
| `-sX` | Xmas scan | FIN + PSH + URG flags |
| `-sA` | ACK scan | Maps firewall rules (filtered vs unfiltered) |
| `-sW` | Window scan | Like ACK, checks TCP window size |
| `-sM` | Maimon scan | FIN/ACK flags |

Null/FIN/Xmas rely on RFC behavior: closed ports reply RST, open ports stay silent (so results show `open|filtered`). They don't work reliably against Windows hosts.

### 5.5 Other

| Flag | Purpose |
|---|---|
| `-sI <zombie>` | Idle scan (uses a third-party host to hide the scanner's IP) |
| `-sO` | IP protocol scan (which protocols are supported: ICMP, TCP, UDP...) |
| `-b <ftp relay>` | FTP bounce scan (mostly obsolete) |

---

## 6. Port Specification

```bash
nmap -p 80 <target>              # single port
nmap -p 22,80,443 <target>       # list
nmap -p 1-1000 <target>          # range
nmap -p- <target>                # ALL 65535 ports
nmap -F <target>                 # Fast: top 100 ports
nmap --top-ports 200 <target>    # top N most common ports
nmap -p T:80,U:53 <target>       # specific protocol
```

> Default Nmap scans the **top 1000** most common ports, not all 65535.

---

## 7. Service & Version Detection (`-sV`)

```bash
nmap -sV 192.168.1.10
nmap -sV --version-intensity 5 192.168.1.10   # 0 (light) to 9 (try everything)
nmap -sV --version-light 192.168.1.10
nmap -sV --version-all 192.168.1.10
```

Output example:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.52
```

Why it matters: version numbers let you look up known CVEs.

---

## 8. OS Detection (`-O`)

```bash
sudo nmap -O 192.168.1.10
sudo nmap -O --osscan-guess 192.168.1.10
```

- Works by fingerprinting TCP/IP stack behavior.
- Needs at least one open and one closed port for best accuracy.
- Results are **guesses**, not guarantees.

---

## 9. Aggressive & Combined Scans

```bash
nmap -A 192.168.1.10
```

`-A` = `-sV` + `-O` + `-sC` (default scripts) + `--traceroute`. Thorough but **noisy**. Avoid on production or in stealth-sensitive work.

---

## 10. Timing & Performance

| Template | Name | Use |
|---|---|---|
| `-T0` | Paranoid | IDS evasion, extremely slow |
| `-T1` | Sneaky | IDS evasion, very slow |
| `-T2` | Polite | Reduces bandwidth/target load |
| `-T3` | Normal | **Default** |
| `-T4` | Aggressive | Fast, good for reliable networks / labs |
| `-T5` | Insane | Very fast, may miss results or crash flaky hosts |

Other tuning options:

```bash
--min-rate 1000          # send at least 1000 packets/sec
--max-retries 2          # limit retransmissions
--host-timeout 5m        # give up on slow hosts
--max-rtt-timeout 500ms
```

---

## 11. Nmap Scripting Engine (NSE)

Scripts live in `/usr/share/nmap/scripts/` (Linux). Written in Lua.

```bash
nmap -sC <target>                          # default scripts (same as --script=default)
nmap --script=vuln <target>                # vulnerability category
nmap --script=http-title <target>          # one script
nmap --script="http-*" <target>            # wildcard
nmap --script="default and safe" <target>  # boolean expressions
nmap --script-help http-enum               # help for a script
nmap --script-updatedb                     # refresh script database
```

### Script Categories

`auth`, `broadcast`, `brute`, `default`, `discovery`, `dos`, `exploit`, `external`, `fuzzer`, `intrusive`, `malware`, `safe`, `version`, `vuln`

> ⚠️ `brute`, `dos`, `exploit`, and `intrusive` scripts can disrupt services. Never run them without explicit authorization.

### Useful Scripts

| Script | What it does |
|---|---|
| `http-title` | Grabs page title |
| `http-enum` | Enumerates common web directories |
| `http-headers` | Shows HTTP headers |
| `ssl-cert` | Shows TLS certificate info |
| `ssl-enum-ciphers` | Lists supported TLS ciphers and rates them |
| `smb-os-discovery` | Gets OS/domain info from SMB |
| `smb-enum-shares` | Lists SMB shares |
| `ftp-anon` | Checks for anonymous FTP login |
| `dns-brute` | Brute-forces subdomains |
| `banner` | Grabs service banners |
| `vulners` | Looks up CVEs for detected versions (needs `-sV`) |

Passing arguments:

```bash
nmap --script http-enum --script-args http-enum.basepath=/app/ <target>
```

---

## 12. Output Formats

```bash
nmap -oN scan.txt <target>     # Normal (human readable)
nmap -oX scan.xml <target>     # XML (for tools / parsing)
nmap -oG scan.gnmap <target>   # Grepable (deprecated but handy)
nmap -oA scan <target>         # ALL three formats at once (recommended)
```

Verbosity and debugging:

```bash
-v / -vv       # more verbose
-d / -dd       # debugging
--reason       # show WHY a port is in a given state
--open         # show only open ports
--packet-trace # show every packet sent/received
```

Resume an interrupted scan: `nmap --resume scan.gnmap`

Convert XML to HTML report: `xsltproc scan.xml -o scan.html`

---

## 13. Firewall / IDS Evasion Concepts

Understanding these helps **defenders** know what to look for. Use only in authorized tests.

| Flag | Idea |
|---|---|
| `-f` / `--mtu <n>` | Fragment packets |
| `-D RND:5` | Decoy scan: mix in fake source IPs |
| `-S <ip>` | Spoof source address |
| `--source-port 53` | Use a trusted-looking source port |
| `--data-length 25` | Append random data to packets |
| `--randomize-hosts` | Shuffle target order |
| `--spoof-mac <mac>` | Spoof MAC address (local network) |
| `-T1` / `-T0` | Slow down to stay under detection thresholds |
| `-Pn` | Don't ping first (when ICMP is blocked) |

Modern IDS/IPS (Snort, Suricata, Zeek) detect many of these. Evasion is never guaranteed.

---

## 14. Common Command Recipes

```bash
# 1. Find live hosts on my network
sudo nmap -sn 192.168.1.0/24

# 2. Quick scan of top 100 ports
nmap -F 192.168.1.10

# 3. Full TCP port scan, then dig into open ports
sudo nmap -p- --min-rate 1000 -T4 -oA full 192.168.1.10
sudo nmap -sC -sV -p 22,80,443 -oA detail 192.168.1.10

# 4. Standard recon scan
sudo nmap -sC -sV -O -oA recon 192.168.1.10

# 5. Vulnerability scripts on a web server
nmap -sV --script vuln -p 80,443 192.168.1.10

# 6. Top UDP ports
sudo nmap -sU --top-ports 50 192.168.1.10

# 7. Scan from a list, save everything
sudo nmap -iL targets.txt -sV -oA results

# 8. Scan when host blocks ping
nmap -Pn -sV 192.168.1.10

# 9. Check TLS config
nmap --script ssl-enum-ciphers -p 443 example.com

# 10. Show only open ports with reasons
nmap --open --reason 192.168.1.10
```

---

## 15. Defender / SOC Perspective

How scans look from the blue-team side:

- **Many SYNs to many ports from one source** → classic port scan signature.
- **Sequential or randomized port access** in a short window.
- **Many hosts probed on one port** → horizontal scan (looking for a specific service).
- **Odd TCP flags** (NULL, FIN, Xmas) → almost always scanning/recon.
- **Nmap's default User-Agent** in NSE HTTP scripts: `Nmap Scripting Engine`.

Where to detect: firewall logs, IDS/IPS alerts (Snort/Suricata rules), Zeek `conn.log`, SIEM correlation rules (e.g., one IP hitting more than N ports in M seconds).

Hardening: close unused ports, use host/network firewalls, rate-limit ICMP, deploy IDS, keep service banners minimal, and patch regularly.

---

## 16. Cheat Sheet

| Goal | Command |
|---|---|
| Live hosts | `nmap -sn <cidr>` |
| Default scan | `nmap <target>` |
| All ports | `nmap -p- <target>` |
| Version detection | `nmap -sV <target>` |
| OS detection | `sudo nmap -O <target>` |
| Default scripts | `nmap -sC <target>` |
| Aggressive | `nmap -A <target>` |
| UDP | `sudo nmap -sU <target>` |
| No ping | `nmap -Pn <target>` |
| Fast | `nmap -F <target>` |
| Save all outputs | `nmap -oA name <target>` |
| Vuln scripts | `nmap --script vuln <target>` |

---

## 17. Practice Targets

- `scanme.nmap.org` (official test host, keep scans light)
- Your own VMs: Metasploitable 2, DVWA, Ubuntu server in VirtualBox/VMware
- TryHackMe: *Nmap* room, *Network Services* rooms
- Hack The Box starting-point machines

## 18. References

- Official site: https://nmap.org
- Reference guide (man page): https://nmap.org/book/man.html
- NSE script docs: https://nmap.org/nsedoc/
- Book: *Nmap Network Scanning* by Gordon "Fyodor" Lyon (free online)

# 🔍 Linux Breach Investigation

> A end-to-end Digital Forensics & Incident Response (DFIR) investigation of a compromised Linux web server — reconstructing a full attack chain across six evidence domains.

---

## 📌 Overview

A Linux web server was compromised on **November 14, 2025**. This repository documents a complete forensic investigation conducted across six challenge domains, correlating evidence from logs, memory, network traffic, disk images, and malware samples to reconstruct the full attack narrative.

This investigation demonstrates:

- **Blue team thinking** — systematic evidence collection, correlation, and timeline reconstruction
- **Attacker visibility** — understanding every step of the attack from initial recon to rootkit deployment
- **Post-exploitation analysis** — tracing persistence mechanisms, C2 channels, and data exfiltration
- **Tool proficiency** — hands-on use of industry-standard DFIR tools

---

## 🧑‍💻 Analyst

**Arnold Mavhezha**
MS Cybersecurity — Yeshiva University
Penetration Tester | Security Engineer | DFIR Analyst
[mavhezha.com](https://mavhezha.com)

---

## 🗂️ Investigation Domains

| # | Domain | Tools | Status |
|---|--------|-------|--------|
| 01 | [Log Parsing](challenges/01-log-parsing/) | grep, awk, sort, uniq | ✅ 6/6 flags |
| 02 | [Memory Forensics](challenges/02-memory-forensics/) | Volatility3, strings | ✅ 6/6 flags |
| 03 | [Network Forensics](challenges/03-network-forensics/) | tshark, base64 | ✅ 5/5 flags |
| 04 | [Disk Forensics](challenges/04-disk-forensics/) | foremost, mount, strings | ✅ 5/5 flags |
| 05 | [Malware Triage](challenges/05-malware-triage/) | strings, grep, cat | ✅ 5/5 flags |
| 06 | [Incident Timeline](challenges/06-incident-timeline/) | All of the above | ✅ 2/3 flags |

**Total: 29/30 flags captured**

---

## ⚔️ Attack Summary

The attacker (`198.51.100.47`) conducted a targeted attack against a Linux web server over approximately one hour:

```
02:55 → Reconnaissance       DirBuster directory scan + port scan
03:12 → Initial Access       SSH brute force (83 attempts)
03:30 → Exploitation         PHP web shell uploaded (/uploads/shell.php)
03:31 → Privilege Escalation sudo /bin/bash → root
03:32 → Execution            Cobalt Strike beacon downloaded and executed
03:33 → Credential Access    cat /etc/shadow via reverse shell
03:33 → Persistence          Backdoor user svc-backup created
03:34 → Persistence          SSH key injected into authorized_keys
03:45 → Persistence          Cron job: */5 * * * * /tmp/.hidden/beacon.sh
03:47 → Command & Control    Encrypted C2 channel to 203.0.113.99:4444 (AES-256)
03:50 → Defense Evasion      Rootkit loaded (rootkit_mod) hiding PID 31337
03:55 → Exfiltration         DNS tunneling to evil-c2.example.com
```

---

## 🛠️ Tools Used

| Tool | Category | Purpose |
|------|----------|---------|
| `grep`, `awk`, `sort`, `uniq` | Log Analysis | Parse and filter log files |
| `Volatility3` | Memory Forensics | Analyze RAM dumps |
| `strings` | Binary Analysis | Extract readable text from binaries |
| `tshark` | Network Forensics | Analyze packet captures |
| `foremost` | Disk Forensics | Carve files from raw disk images |
| `mount` | Disk Forensics | Mount disk images for analysis |
| `base64` | Data Transfer | Encode/decode binary files |
| `sha256sum` | Integrity | Hash files for verification |
| `yara` | Malware Detection | Pattern-based malware identification |

---

## 🔑 Key Findings

**Attacker Infrastructure:**
- External IP: `198.51.100.47`
- C2 Server: `203.0.113.99` / `evil-c2.example.com`
- C2 Port: `4444` (reverse shell), `443` (beacon)
- Payload server: `http://198.51.100.47/payload.elf`

**Compromised System:**
- Internal IP: `10.0.1.50`
- Hostname: `webserver`
- OS: Linux (Ubuntu)

**Malware:**
- Beacon: `/tmp/.update/beacon` (disguised as `kworker-update`, PID 31337)
- Encryption: AES-256
- User-Agent: `Mozilla/5.0 (compatible; beacon/2.0)`
- Rootkit: `rootkit_mod.ko`

**Persistence Mechanisms:**
- Backdoor user: `svc-backup` (password: `Sup3rS3cret!`)
- SSH key: `/home/svc-backup/.ssh/authorized_keys`
- Cron job: `*/5 * * * * /tmp/.hidden/beacon.sh`

**Exfiltrated Data:**
- `/etc/shadow` — password hashes
- `/home/*/.ssh/id_rsa` — SSH private keys
- `/home/*/.bash_history` — command histories
- `/documents/.secret/stolen-data.csv` — stolen data on USB

---

## 📁 Repository Structure

```
linux-breach-investigation/
├── README.md                        ← This file
├── timeline.txt                     ← Canonical attack timeline
├── methodology.md                   ← DFIR investigation methodology
├── commands-used.md                 ← All commands with explanations
├── challenges/
│   ├── 01-log-parsing/
│   │   ├── 01-log-parsing-writeup.md
│   │   └── screenshots/
│   ├── 02-memory-forensics/
│   │   ├── 02-memory-forensics-writeup.md
│   │   └── screenshots/
│   ├── 03-network-forensics/
│   │   ├── 03-network-forensics-writeup.md
│   │   └── screenshots/
│   ├── 04-disk-forensics/
│   │   ├── 04-disk-forensics-writeup.md
│   │   └── screenshots/
│   ├── 05-malware-triage/
│   │   ├── 05-malware-triage-writeup.md
│   │   └── screenshots/
│   └── 06-incident-timeline/
│       ├── 06-incident-timeline-writeup.md
│       └── screenshots/
└── screenshots/
```

---

## 📜 Disclaimer

This investigation was conducted in a controlled lab environment provided for educational purposes.
All techniques documented here are for **defensive security** and **educational use only**.
Never apply these techniques against systems you do not have explicit permission to analyze.

---

*Investigation Date: March–April 2026*
*Analyst: Arnold Mavhezha | mavhezha.com*

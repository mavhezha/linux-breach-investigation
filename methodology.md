# DFIR Investigation Methodology

**Analyst:** Arnold Mavhezha
**Investigation:** Linux Breach — November 14, 2025
**Approach:** Evidence-first, hypothesis-driven forensic investigation

---

## Core Principles

Every forensic investigation I conduct follows these principles:

**1. Preserve Before Analyzing**
Never modify evidence. Always mount disk images read-only (`mount -o ro,loop`). Work on copies, never originals. Hash evidence files before and after analysis to verify integrity.

**2. Follow the Timestamps**
Every finding leads to the next. Each event has a timestamp that tells you when to look next. Build the timeline incrementally — one event points to the next.

**3. Cross-Reference Everything**
No single evidence source tells the whole story. A finding in `auth.log` should be corroborated by `syslog`. A network connection in `capture.pcap` should match the C2 IP from memory forensics. Correlation builds confidence.

**4. Document As You Go**
Screenshots are taken immediately after every finding. Commands are recorded with their purpose. Even failed attempts are documented — understanding what did not work is as valuable as finding the correct answer.

**5. Think Like the Attacker**
Understanding the attacker's goals at each phase helps predict what evidence to look for next. After seeing a web shell upload, you look for command execution. After command execution, you look for persistence.

---

## Investigation Workflow

### Phase 1 — Scoping
```
1. Read all available README and scenario files
2. List all evidence files and understand what each contains
3. Identify the known indicators (attacker IP, victim IP, timeframe)
4. Plan the investigation order — start with logs, then memory, then network
```

### Phase 2 — Log Analysis
```
1. Identify attacker IP from failed authentication patterns
2. Trace web activity from access logs (uploads, shell execution)
3. Follow timestamps across multiple log files
4. Cross-reference auth.log + apache-access.log + syslog + kern.log
```

### Phase 3 — Memory Forensics
```
1. Examine process tree for anomalies (wrong parent, suspicious names)
2. Check cmdline output for real binary paths and C2 arguments
3. Review bash history for exact attacker commands
4. Use strings on memory samples to recover credentials and artifacts
```

### Phase 4 — Network Forensics
```
1. Filter DNS traffic to identify C2 domains
2. Identify compromised hosts by traffic volume and communication patterns
3. Count SYN packets to identify port scanning activity
4. Follow TCP streams to reconstruct reverse shell sessions
5. Decode base64 DNS subdomains to identify exfiltration content
```

### Phase 5 — Disk Forensics
```
1. Mount disk images read-only
2. Perform recursive listing including hidden files (ls -laR)
3. Use foremost to carve deleted/hidden files from raw images
4. Use strings on raw images to recover deleted file contents
5. Look for cleanup scripts and staged exfiltration archives
```

### Phase 6 — Malware Analysis
```
1. Never execute suspicious files
2. Use strings to extract hardcoded IPs, URLs, and credentials
3. Examine VBA macros for auto-execution, encoded commands, and download URLs
4. Look for sandbox evasion checks
5. Correlate malware artifacts with network and log evidence
```

### Phase 7 — Timeline Construction
```
1. Collect all confirmed timestamps from every evidence source
2. Standardize to ISO 8601 format (YYYY-MM-DDTHH:MM:SSZ)
3. Sort chronologically
4. Map each event to a MITRE ATT&CK technique
5. Write the narrative — tell the story of the attack from start to finish
```

---

## Evidence Correlation Map

```
auth.log ─────────────────┐
apache-access.log ─────────┤
syslog ────────────────────┤──► Timeline
kern.log ──────────────────┤
ids-alerts.json ───────────┤
                           │
capture.pcap ──────────────┤
pslist/cmdline ────────────┤
bash-history ──────────────┤
                           │
disk images ───────────────┤
malware samples ───────────┘
```

Every evidence source contributes a piece of the puzzle. No single source is sufficient on its own.

---

## MITRE ATT&CK Mapping

| Technique | ID | Evidence Source |
|-----------|-----|----------------|
| Active Scanning | T1595 | apache-access.log, capture.pcap |
| Brute Force | T1110 | auth.log |
| Web Shell | T1505.003 | apache-access.log |
| Command and Scripting Interpreter | T1059 | bash-history, apache-access.log |
| Create Account | T1136 | auth.log |
| SSH Authorized Keys | T1098.004 | auth.log |
| Scheduled Task/Cron | T1053.003 | syslog |
| Rootkit | T1014 | kern.log, pslist |
| Exfiltration Over DNS | T1048.003 | capture.pcap, ids-alerts.json |
| OS Credential Dumping | T1003 | bash-history, apache-access.log |

---

## Lessons Learned

**From Log Analysis:**
Classic log analysis one-liners (`grep | awk | sort | uniq -c | sort -rn`) quickly surface anomalies like brute force IPs. Always search across multiple log files simultaneously.

**From Memory Forensics:**
Process trees reveal disguised malware — always check parent-child relationships. Bash history in RAM captures the attacker's exact commands. Cross-reference `strings` output against other sources to filter memory artifacts.

**From Network Forensics:**
DNS traffic is a goldmine — it reveals both C2 infrastructure and covert exfiltration channels. TCP stream reconstruction is essential for understanding reverse shell sessions.

**From Disk Forensics:**
Deleting files does not erase data. `strings` on raw images recovers deleted content. `foremost` carves files even when the filesystem has no record of them.

**From Malware Triage:**
Static analysis is safe and powerful. Hardcoded C2 addresses, User-Agent strings, and encoded payloads are all recoverable without execution. Cross-reference malware artifacts with network and log evidence.

**From Incident Timeline:**
Timestamp format consistency is critical when hashing timelines. Always standardize to ISO 8601. Every challenge tells the same story from a different angle — correlation is everything.

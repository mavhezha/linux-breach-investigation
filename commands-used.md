# Commands Used — Linux Breach Investigation

> Every command used across all six challenge domains, with explanations.
> Built as a reference for analysts and reviewers.

---

## Log Analysis Commands

### Find Brute Force IP
```bash
sudo grep "Failed password" auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head
```
Searches auth.log for failed SSH attempts, extracts the source IP (field 11), counts occurrences per IP, and sorts highest first. A single IP with significantly more failures than others is the brute force source.

### Find Web Shell Upload
```bash
sudo grep -i "\.php" apache-access.log | grep "POST"
```
Filters apache access log for PHP file requests using POST method. File uploads use POST — this reveals any PHP shells uploaded to the server.

### Find User Creation Events
```bash
sudo grep -i "useradd\|adduser\|new user" auth.log
```
Searches auth.log for any user creation events using OR patterns (`\|`). Attackers create backdoor accounts disguised as legitimate service accounts.

### Find Crontab Activity
```bash
sudo grep -i "svc-backup\|beacon\|connect\|curl\|wget" auth.log syslog
```
Searches both auth.log and syslog simultaneously for attacker activity. Passing multiple files to grep searches all of them in one command.

### Find C2 Callback in Firewall Logs
```bash
sudo grep "Nov 14 03:4" syslog
```
Filters syslog for a specific time window. UFW firewall logs show outbound connections — a compromised host connecting outbound to a high port is a C2 callback indicator.

### Find Kernel Module Activity
```bash
sudo grep -i "module\|insmod\|modprobe" kern.log
```
Searches kern.log for kernel module loading events. Attackers load malicious kernel modules (rootkits) to hide processes and maintain persistence.

### Read Suspicious Crontab
```bash
sudo cat suspicious-cron.log
```
Reads the exported crontab entries. The cron schedule format is: `minute hour day month weekday command`. `*/5 * * * *` means every 5 minutes.

---

## Memory Forensics Commands

### Read Process Tree
```bash
cat evidence/pstree-output.txt
```
Displays the parent-child process hierarchy. Malicious processes disguise themselves as legitimate system processes but appear under the wrong parent — `kworker` threads should be children of `kthreadd`, not spawned from `bash`.

### Read Command Line Arguments
```bash
cat evidence/cmdline-output.txt
```
Shows the full command line used to launch each process. Reveals real binary paths even when process names are disguised, and exposes C2 addresses and encryption flags passed as arguments.

### Read Bash History from Memory
```bash
cat evidence/bash-history-output.txt
```
Displays bash commands recovered from RAM. Memory captures commands even if `.bash_history` was deleted on disk. This revealed the complete attacker playbook.

### Extract Strings from Memory Dump
```bash
sudo strings practice.raw | grep -i "password"
```
Extracts all readable text from a raw memory dump and filters for password-related strings. Memory can contain decrypted credentials that were never written to disk.

---

## Network Forensics Commands

### Filter DNS Traffic
```bash
sudo tshark -r capture.pcap -Y "dns"
```
Reads a PCAP file and displays only DNS packets. DNS traffic reveals both C2 domain resolution and covert data exfiltration via encoded subdomains.

### Get IP Endpoint Statistics
```bash
sudo tshark -r capture.pcap -q -z endpoints,ip
```
Generates a summary of all IP addresses in the capture with traffic volumes. The compromised host stands out by having the highest volume and direct C2 communication.

### Count Scanned Ports
```bash
sudo tshark -r capture.pcap -Y "ip.src == 198.51.100.47 && tcp.flags.syn == 1 && tcp.flags.ack == 0" -T fields -e tcp.dstport | sort -n | uniq | wc -l
```
Filters for TCP SYN packets (port scan signature) from the attacker's IP, extracts destination ports, removes duplicates, and counts them. SYN without ACK = port scan.

### Decode Base64 DNS Exfiltration
```bash
echo "c3RvbGVuX2NyZWRlbnRpYWxz" | base64 -d
```
Decodes a base64-encoded string found in a DNS subdomain. Attackers encode stolen data in DNS queries to exfiltrate it through firewalls that allow DNS traffic.

### Find Reverse Shell Stream
```bash
sudo tshark -r capture.pcap -T fields -e tcp.stream -Y "tcp.port == 4444" | sort -n | uniq
```
Identifies all TCP streams involving the C2 port (4444). Each unique stream number represents a separate connection session.

### Follow TCP Stream
```bash
sudo tshark -r capture.pcap -q -z follow,tcp,ascii,190
```
Reconstructs the full conversation of TCP stream 190 in ASCII format. This reveals the exact commands typed by the attacker during the reverse shell session.

---

## Disk Forensics Commands

### Mount Disk Image Read-Only
```bash
sudo mount -o ro,loop suspicious.dd /mnt
```
Mounts a raw disk image as a filesystem. `ro` = read-only (preserves evidence integrity). `loop` = loopback device (treats the file as a block device). Always mount read-only.

### Recursive Directory Listing Including Hidden Files
```bash
ls -laR /mnt
```
Lists all files recursively (`R`) with full details (`l`) including hidden dotfiles (`a`). Hidden directories start with `.` and are invisible without the `-a` flag.

### Carve Files from Raw Disk Image
```bash
sudo foremost -t png,jpg,bmp,gif -i suspicious.dd -o /tmp/carved-sus
```
Carves files from a raw disk image by scanning for file headers and footers (magic bytes). Recovers files even when the filesystem has no record of them. `-t` specifies file types to carve.

### Recover Deleted File Contents
```bash
strings deleted-files.img | grep -i "pass\|admin\|user\|cred"
```
Extracts readable text from a raw disk image and filters for credential keywords. Deleted files remain in unallocated disk space until overwritten — `strings` recovers their contents.

### Recover Deleted File with Context
```bash
strings deleted-files.img | grep -i -A5 -B5 "exfil\|2025\|plan"
```
Same as above but shows 5 lines before (`B5`) and after (`A5`) each match — providing context around recovered fragments.

### Transfer Binary File via Base64
```bash
# On remote server — encode
base64 /tmp/carved-sus/png/00012416.png

# On local machine — decode
cat > /tmp/flag.b64 << 'EOF'
<BASE64_DATA>
EOF
base64 -d /tmp/flag.b64 > recovered-image.png
```
Encodes a binary file to base64 text for transfer from a restricted environment with no file transfer tools. The heredoc (`<< 'EOF'`) avoids issues with special characters in the base64 string.

---

## Malware Triage Commands

### Extract Strings from ELF Binary
```bash
strings sample.elf | grep -iE "([0-9]{1,3}\.){3}[0-9]{1,3}|http|c2|beacon|connect"
```
Extracts readable strings from an ELF binary and filters for IP addresses (via regex) and C2-related keywords. Hardcoded C2 addresses are a common finding in malware samples.

### Find Embedded Flags
```bash
strings sample.elf | grep FLAG
```
Simple but effective — extracts all strings and filters for the flag format. Debug strings and test artifacts are often left in binaries by developers.

### Find User-Agent Strings
```bash
strings sample.elf | grep -i "user-agent\|mozilla\|browser"
```
Searches for HTTP User-Agent strings embedded in the binary. Malware masquerades as legitimate browsers to blend in with normal web traffic.

### Read VBA Macro Source
```bash
cat macro-source.vba
```
Reads VBA macro source code directly. Reveals auto-execution methods, encoded PowerShell commands, download URLs, and sandbox evasion checks — all without executing the file.

---

## General Utility Commands

### Submit Flag
```bash
sudo /lab/tools/submit-flag.sh <question_number> "FLAG{value}"
# For flags containing ! use single quotes
sudo /lab/tools/submit-flag.sh 2.6 'FLAG{Sup3rS3cret!}'
```
Submits a flag to the scoring system. Single quotes prevent bash from interpreting special characters like `!`.

### Check Scoreboard
```bash
/lab/scoreboard.sh
```
Displays current challenge completion status and score.

### Hash a File
```bash
sha256sum filename.txt
```
Generates the SHA-256 hash of a file. Used for integrity verification and timeline hashing challenges.

### Identify File Type
```bash
file suspicious_file
```
Identifies a file's true type by examining its magic bytes — not its extension. A file named `.jpg` might actually be an ELF binary.

### Decode Base64 String
```bash
echo "BASE64STRING" | base64 -d
```
Decodes a base64-encoded string. Used for DNS exfiltration decoding and PowerShell encoded command analysis.

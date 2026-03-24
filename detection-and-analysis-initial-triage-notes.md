# Initial Triage (Detection & Analysis) – Notes

## Purpose of Initial Triage

Initial triage is the **first forensic inspection of a compromised system while it is still running**.

Goals:

• Identify signs of compromise
• Capture volatile evidence
• Preserve artifacts for later analysis
• Determine scope of intrusion

Important rule:

⚠️ **Collect evidence first. Analyze later.**

All collected artifacts are saved to a directory:

```
$datapath
```

This keeps evidence organized.

---

# 1. Start Documentation

## Start PowerShell Transcript

Command

```powershell
Start-Transcript -Path $datapath\transcript.txt
```

### What it does

Records **every command and output** executed during triage.

### Why we use it

Creates a **forensic audit trail**.

Investigators must document every action.

---

## Captain's Log Entry

Command

```powershell
echo "captains_log: starting triage $(Get-Date)" >> $datapath\captains_log.txt
```

### What it does

Adds timestamped notes.

### Why we use it

Documents important actions during investigation.

---

# 2. Basic System Information

## Tool

PowerShell / Windows commands

### Command

```powershell
systeminfo
```

or

```powershell
Get-ComputerInfo
```

### What it does

Displays:

• OS version
• patch level
• hostname
• boot time
• hardware info
• domain membership

### Why we use it

Gives **context about the machine**.

Example evidence:

• outdated OS
• unpatched vulnerabilities
• suspicious system uptime

---

# 3. Running Process Investigation

Processes are the **most important triage artifact** because malware must run as a process.

---

## List Running Processes

Command

```powershell
tasklist
```

### What it does

Lists:

• running processes
• process IDs (PID)
• memory usage

### Why we use it

Find suspicious processes.

Example indicators:

• strange names
• high CPU usage
• unknown executables

---

## Advanced Process Analysis

Tool: **Sysinternals Process Explorer**

Application:

```
procexp.exe
```

### What it does

Shows:

• parent-child process relationships
• loaded DLLs
• network connections
• digital signatures

### Why we use it

Helps detect:

• process injection
• malware spawning processes
• suspicious parent relationships

Example finding:

```
iamironcatwashere.exe
```

Suspicious unknown process.

---

# 4. Network Connection Investigation

Attackers communicate with **command and control servers**.

---

## View Active Network Connections

Command

```powershell
netstat -ano
```

### What it does

Displays:

• open ports
• remote IP addresses
• connection states
• PID of process using connection

Example output

```
TCP 192.168.1.10:49821 -> 52.251.113.157
```

### Why we use it

Identifies **external connections**.

Example finding:

```
hello.iamironcat.com
```

Suspicious remote server.

---

## Save Network Connections

Command

```powershell
netstat -ano > $datapath\netconn.txt
```

### Evidence found

• attacker IP addresses
• malware communication
• backdoor connections

---

# 5. DNS Cache Investigation

## Command

```powershell
ipconfig /displaydns
```

### What it does

Shows **recent DNS lookups**.

Example:

```
hello.iamironcat.com
```

### Why we use it

DNS reveals:

• malware domains
• command servers
• phishing sites visited

---

# 6. ARP Cache Investigation

## Command

```powershell
arp -a
```

### What it does

Shows devices the computer recently communicated with on the **local network**.

### Why we use it

Detect:

• lateral movement
• nearby infected machines

Example output

```
192.168.1.5
192.168.1.20
```

---

# 7. Routing Table

## Command

```powershell
Get-NetRoute
```

### What it does

Shows network routes.

### Why we use it

Malware sometimes **manipulates routes** to redirect traffic.

---

# 8. Firewall Rules

## Command

```powershell
Get-NetFirewallRule
```

### What it does

Lists firewall rules.

### Why we use it

Attackers may add rules to:

• allow malware traffic
• bypass security

Example finding:

```
Allow TCP 8080 inbound
```

---

# 9. User Account Investigation

Attackers often create **new user accounts**.

---

## List Local Users

Command

```powershell
net user
```

### What it does

Displays all local accounts.

### Evidence

• new admin users
• suspicious usernames

---

## Check Administrator Group

Command

```powershell
net localgroup administrators
```

### What it does

Shows who has admin privileges.

### Why we use it

Privilege escalation detection.

---

# 10. Logged-In Users

## Tool

Sysinternals

```
psloggedon.exe
```

### What it does

Shows users logged onto the system.

### Evidence

• attacker remote logins
• compromised accounts

---

## Logon Sessions

Tool

```
logonsessions64.exe
```

### What it does

Shows:

• session IDs
• authentication type
• login times

Useful for investigating **Windows event logs later**.

---

# 11. Network Shares

## Command

```powershell
net use
```

### What it does

Shows mapped drives and shares.

### Why we use it

Ransomware spreads through **network shares**.

---

# 12. Service Investigation

Malware frequently installs itself as a **Windows service**.

---

## Command

```powershell
cmd /c sc query
```

### What it does

Lists system services.

### Evidence

• suspicious service names
• unknown executables

---

## Alternative Method

Command

```powershell
Get-WmiObject Win32_Service
```

---

# 13. Persistence Detection

Persistence allows malware to **survive reboot**.

---

## Tool

Sysinternals **Autoruns**

```
autoruns.exe
```

### What it checks

• startup programs
• scheduled tasks
• services
• drivers
• registry run keys

### Why we use it

Find malware persistence locations.

---

# 14. Scheduled Tasks

## Command

```powershell
schtasks /query /fo LIST /v
```

### What it does

Lists scheduled tasks.

### Evidence

Attackers often schedule tasks to:

• run malware
• reconnect to C2

---

# 15. DLL Analysis

Processes load **DLL modules**.

## Tool

```
listdlls.exe
```

### What it does

Shows DLLs loaded by processes.

### Why we use it

Detects:

• malicious DLL injection
• unknown libraries

---

# 16. File System Metadata

## Tool

```
ntfsinfo.exe
```

### Command

```
ntfsinfo C:
```

### What it does

Displays NTFS filesystem details.

Useful for:

• forensic timeline analysis
• metadata inspection

---

# 17. Event Log Collection

Windows logs contain **authentication and security events**.

---

## Command

```powershell
wevtutil qe Security
```

### What it does

Queries security event log.

Evidence includes:

• logins
• failed logins
• privilege escalation
• service installation

---

## Copy Event Logs

Command

```powershell
copy-item C:\Windows\System32\winevt\Logs\* $datapath\winevent_logs
```

### Why we use it

Preserves **raw EVTX logs** for forensic tools.

---

# 18. System Log Files

Some logs are stored in directories.

Example

```
C:\Windows\System32\LogFiles
```

### Command

```powershell
copy-item C:\Windows\System32\LogFiles\* $datapath\LogFiles
```

Evidence includes:

• firewall logs
• IIS logs
• system events

---

# 19. Network Traffic Capture

Capturing traffic reveals **attacker communication**.

---

## Tool

Wireshark

### Example Filter

```
tcp.port == 8080
```

Used to detect **web shell activity**.

Example:

```
dir
```

Command sent to web shell.

---

## Filter by IP

Example

```
ip.addr == 52.251.113.157
```

Used to identify traffic to:

```
hello.iamironcat.com
```

---

# 20. Portable Packet Capture

Wireshark requires installation.

Instead we use **RawCap**.

---

## Tool

```
rawcap.exe
```

### Identify interfaces

```
rawcap.exe
```

Example output

```
Interface 0
```

---

## Capture Packets

Command

```
rawcap.exe 0 $datapath\network_capture.pcap
```

### What it does

Captures live network traffic.

### Output

```
network_capture.pcap
```

Used later for analysis in Wireshark.

---

# Evidence Types Collected

During triage we collect:

### System data

• OS version
• host configuration

### Process artifacts

• running processes
• DLL modules

### Network artifacts

• active connections
• DNS queries
• ARP entries

### User artifacts

• accounts
• login sessions

### Persistence mechanisms

• services
• scheduled tasks
• startup entries

### Logs

• Windows security logs
• firewall logs

### Traffic captures

• PCAP files for network analysis

---

# Final Step After Triage

Once initial triage finishes:

Next step is **full forensic analysis**.

This includes:

• memory analysis
• disk imaging
• timeline analysis
• malware reverse engineering

---

# One Sentence Summary

Initial triage quickly collects **volatile forensic artifacts from a live system** so investigators can later reconstruct the attacker’s activity.



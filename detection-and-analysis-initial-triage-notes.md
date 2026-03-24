```markdown
# 🛡️ Host Analysis – Notes (Incident Response Pathway)

Network connection → Process → Memory → Payload → File → Logs → Timeline → Root cause

````

---

# 🧰 Tools Used in This Course

## 1. Velociraptor
**What it is:**
- A DFIR (Digital Forensics & Incident Response) tool

**What it does:**
- Collects system data remotely
- Runs forensic queries
- Builds timelines
- Investigates endpoints

**Why we use it:**
- Central tool for gathering host evidence quickly

---

## 2. Volatility
**What it is:**
- Memory forensics framework

**What it does:**
- Analyzes RAM (memory dumps)

**Why we use it:**
- Attackers often run malware in memory only (fileless attacks)

---

## 3. CyberChef
**What it is:**
- Data decoding tool

**What it does:**
- Decodes Base64, encryption, obfuscation

**Why we use it:**
- Attackers hide payloads using encoding

---

## 4. oletools (oleid, olevba)
**What it is:**
- Malware analysis tools for Office documents

**What it does:**
- Detects and extracts macros

**Why we use it:**
- Many attacks start with malicious Word/Excel files

---

## 5. Windows Event Viewer
**What it is:**
- Built-in Windows logging system

**What it does:**
- Stores logs for logins, processes, RDP, etc.

**Why we use it:**
- Detect attacker activity and movement

---

# 🔍 STEP-BY-STEP HOST ANALYSIS PROCESS

---

# 1. Initial Triage (First Look at the System)

## Goal:
Quickly understand:
- What’s running
- What’s connected
- What looks suspicious

---

## Command: View Network Connections

```bash
netstat -ano
````

### What it does:

* Shows active network connections
* Shows PID (process ID)

### Why we use it:

* Find suspicious connections (possible attacker communication)

### What to look for:

* Unknown IP addresses
* Strange ports
* Internal pivoting traffic

---

## Command: List Processes

```bash
tasklist
```

### What it does:

* Lists all running processes

### Why we use it:

* Identify suspicious programs

---

## Match Process to Network Connection

```bash
tasklist | findstr <PID>
```

### What it does:

* Finds which process owns a connection

### Why we use it:

* Link suspicious traffic → actual process

---

# 2. Identify Suspicious Behavior

### Red flags:

* PowerShell running unexpectedly
* Unknown executables
* High-numbered ports
* Internal system-to-system communication

---

# 3. Memory Analysis (Find Hidden Commands)

## Goal:

Find attacker commands hidden in memory

---

## Command: List Processes in Memory

```bash
volatility -f memory.raw pslist
```

### What it does:

* Lists processes from memory dump

---

## Command: Process Tree

```bash
volatility -f memory.raw pstree
```

### What it does:

* Shows parent-child relationships

### Why important:

* Shows how malware was launched

Example:

```
explorer.exe → powershell.exe
```

---

## Command: View Command Line Arguments

```bash
volatility -f memory.raw cmdline
```

### What it does:

* Shows how processes were executed

### What to look for:

```
powershell -EncodedCommand ...
```

---

## Command: Network Connections in Memory

```bash
volatility -f memory.raw netscan
```

### What it does:

* Shows active connections from memory

### Why:

* Confirms attacker communication

---

# 4. Detect Reverse Shell

## What is happening:

* Victim connects back to attacker

```
Victim → Attacker (C2 server)
```

## Why attackers use it:

* Bypass firewall restrictions
* Maintain control

---

# 5. Decode Obfuscated PowerShell

## Problem:

Command is encoded (Base64)

---

## Solution: CyberChef

### Steps:

1. Paste encoded string
2. Apply:

```
From Base64
→ Decode UTF-16LE
```

---

## Alternative PowerShell Method

```powershell
[System.Text.Encoding]::Unicode.GetString(
[System.Convert]::FromBase64String("ENCODED_STRING"))
```

---

## What you get:

* Real command
* C2 IP address
* Payload behavior

---

# 6. Identify Command & Control (C2)

## What to extract:

* IP addresses
* Ports
* Domains

## Why:

* Used to find other infected systems

---

# 7. File Analysis – Malware Entry Point

## Goal:

Find how the attack started

---

## Command: Check for Macros

```bash
oleid document.doc
```

### What it does:

* Detects if macros exist

---

## Command: Extract Macro Code

```bash
olevba document.doc
```

### What it does:

* Shows macro script

---

## What to look for:

* AutoOpen
* Encoded strings
* PowerShell execution

---

## What is AutoOpen?

```
Macro runs automatically when document is opened
```

---

# 8. Attack Chain (Initial Access)

```
Phishing email
→ User opens document
→ Macro runs
→ PowerShell executes
→ Malware downloads
```

---

# 9. Log Analysis (Find Attacker Activity)

## Open:

```
Event Viewer
```

---

## Security Logs

```
Windows Logs → Security
```

### Important Event IDs:

| ID   | Meaning          |
| ---- | ---------------- |
| 4624 | Successful login |
| 4625 | Failed login     |

---

## What to look for:

* Many failed logins → brute force
* Unknown login locations

---

# 10. RDP Logs

```
Applications and Services Logs
→ Microsoft
→ Windows
→ RemoteDesktopServices
```

### Shows:

* Remote login activity

---

# 11. SMB Logs (Lateral Movement)

```
Microsoft → Windows → SMBClient
Microsoft → Windows → SMBServer
```

### Port:

```
445
```

### Why:

* Used to move between machines

---

# 12. Timeline Analysis (File Activity)

## Tool:

Velociraptor

---

## Module:

```
Windows.Timeline.MFT
```

---

## Filter:

```
WHERE FullPath endswith ".exe"
```

---

## Why:

* Find attacker tools quickly

---

## What to look for:

* New executables
* Downloads
* Replaced files

---

# 13. Identify Attacker Tools

Examples found:

* PoshC2
* PBindSharp

---

## What they do:

* Remote control of system
* Network pivoting

---

# 14. Service Hijacking

## What happened:

Legitimate service replaced with malicious file

---

## Technique:

```
Binary replacement
```

---

## Target example:

```
DarkEnergyControlService.exe
```

---

# 15. Recovery

## Discovery:

Original file found in:

```
Recycle Bin
```

---

## Action:

Restore original file → system works again

---

# 16. Full Attack Timeline

```
1. Phishing email sent
2. User opens malicious document
3. Macro executes automatically
4. PowerShell runs encoded payload
5. Reverse shell established
6. Attacker connects to C2
7. Lateral movement begins
8. Ransomware deployed
9. Critical service replaced
10. System disrupted
11. Original service recovered
```

---

# 🔑 Key Skills Learned

* Process analysis
* Network connection tracing
* Memory forensics
* PowerShell decoding
* Macro malware analysis
* Log investigation
* Timeline reconstruction
* IOC extraction
* Lateral movement detection
* Service hijacking detection

---

# 🧠 Final Rule of Host Analysis

```
Always follow the evidence.
```

Each step leads to the next:

```
Connection → Process → Memory → Payload → File → Logs → Timeline → Root Cause
```

---


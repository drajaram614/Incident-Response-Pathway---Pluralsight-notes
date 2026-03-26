---

# 🛡️ Incident Response – Eradication Phase (Notes)

## 🎯 Goal of Eradication

Completely **remove the attacker and malware** from the environment.

This includes:

1. Blocking attacker access
2. Identifying all infected systems
3. Finding all malware files
4. Removing persistence
5. Preparing for recovery

---

# 1. Core Problem in Eradication

Even after analysis:

❗ **Attacker is STILL inside the network**

Main risks:

### 1. Active Access

* C2 connections allow attacker control

### 2. Passive Access

* Web shells (port 8080)
* Backdoors

### 3. Reinfection

* Malware can re-download from domain

---

# 2. Key Indicators Identified

From previous phases:

### C2 Server

```
172.31.101.111
```

### Malicious Domain

```
hello.iamironcat.com
```

### Persistence

* Web shells on port **8080**
* Malware executable
* Scheduled/automated actions

---

# 3. Final Scoping (VERY IMPORTANT)

Before eradication, you must find:

✅ Every infected device
✅ Every attacker connection
✅ Every malware file

If you miss anything → attacker stays

---

# 4. Network Scoping (Find Infected Hosts)

## Tool Used: `tcpdump`

### Why we use it

* Analyze network traffic (PCAP)
* Find machines talking to attacker

---

## Command: Read PCAP

```
tcpdump -r ext_traffic.pcap
```

### What it does

* Reads captured network traffic

---

## Filter for C2 Server

```
tcpdump -nr ext_traffic.pcap host 172.31.101.111
```

### Why

* Show ONLY traffic to attacker

---

## Extract Internal IPs

```
cut -d " " -f3
```

→ Gets source IP + port

---

## Remove Port

```
cut -d "." -f1-4
```

→ Keeps just IP

---

## Deduplicate

```
sort -n | uniq
```

---

## Final Command (Full Pipeline)

```
tcpdump -nr ext_traffic.pcap host 172.31.101.111 | cut -d " " -f3 | cut -d "." -f1-4 | sort -n | uniq
```

---

## What You Find

* All internal machines talking to C2

👉 These are **compromised hosts**

---

# 5. Web Shell Detection (Internal Access)

## Tool Used: `nmap`

### Why

* Find open ports (persistence)

---

## Command

```
sudo nmap -sS -Pn -p 8080 10.0.0.0/24
```

---

## What Each Part Does

| Flag      | Meaning                         |
| --------- | ------------------------------- |
| `-sS`     | SYN scan (fast, stealth)        |
| `-Pn`     | skip ping (Windows blocks ICMP) |
| `-p 8080` | check web shell port            |
| `/24`     | scan entire subnet              |

---

## Extract Results

```
cat scan.gnmap | grep open
```

```
cat scan.gnmap | grep open | cut -d " " -f2
```

---

## What You Find

* Machines with port **8080 open**

👉 These have **web shells (attacker access)**

---

# 6. File Signature Scoping (Find Malware Files)

## Problem

File names can change → unreliable

---

## Tool: `strings`

### Why

* Extract readable data from malware

---

## Commands

### Basic

```
strings malware.exe
```

### Cleaner output

```
strings malware.exe | more
```

### Filter useful strings

```
strings -n 10 malware.exe
```

### Find long strings

```
strings -n 100 malware.exe
```

---

## What You Find

* Function names
* Compiler info (Go build ID)
* Ransom note text

---

## Example Signatures

* `Go build ID`
* `friendlyLetter`
* `I'm not a trash panda`

👉 Unique identifiers inside malware

---

# 7. YARA (Signature-Based Detection)

## Tool: YARA

### Why

* Detect malware by content, not name

---

## Example Rule

```
rule FluffyKittens
{
    strings:
        $a = "Go build ID"
        $b = "I'm not a trash panda"
        $c = "friendlyLetter"

    condition:
        any of them
}
```

---

## What It Does

* Scans files
* Matches these strings

---

## What You Find

* Malware copies anywhere on disk
* Even if renamed

---

# 8. Endpoint Hunting (Velociraptor)

## Tool: Velociraptor

### Why

* Scan ALL endpoints at once

---

## Artifact Used

```
Windows.Detection.Yara.NTFS
```

---

## Config

* Path: `C:\`
* File type: `.exe`
* Rule: YARA rule

---

## What You Find

Example:

```
C:\Users\Public\fluffykittens.exe
C:\Windows\SysWOW64\ironcatwashere.exe
```

---

## Key Discovery

* Malware replicated itself
* Changed name

👉 Without YARA → missed

---

# 9. Network Control Strategy

## Goal

Block attacker **without breaking business**

---

## Constraints

* Cannot shut down everything
* Some systems must stay online
* Multiple network paths exist

---

# 10. Blocking C2 Traffic

## Method

Firewall / ACL rules

---

## Rule

Block:

```
172.31.101.111:443
```

---

## Why port 443?

* HTTPS traffic (allowed through firewall)
* attacker hides inside normal traffic

---

## Important Concept

Firewalls allow:

```
internal → external connections
```

So attacker uses this to bypass security.

---

# 11. AWS Network Controls

## Tools Used

### Security Groups

* Allow rules only

### Network ACLs

* Allow + DENY rules

---

## Why ACLs?

We need to **block specific traffic**

---

## Example Rule

```
DENY TCP 443 → 172.31.101.111
```

---

## Critical Concept: Rule Order

Correct:

```
90 → DENY C2
100 → ALLOW ALL
```

Wrong:

```
100 → ALLOW ALL
110 → DENY C2 ❌ (never applied)
```

---

# 12. Blocking Malware Domain

## Step 1: Resolve Domain

```
nslookup hello.iamironcat.com
```

---

## Output

```
52.251.113.157
```

---

## Step 2: Block IP

Add firewall rule:

```
DENY ALL → 52.251.113.157
```

---

## Why

Prevents:

* malware download
* reinfection

---

# 13. What Happens After Blocking

## Immediate Effect

* attacker loses access
* C2 connections die
* implants stop working

---

## Even Daisy-Chained Hosts Fail

Because:

* they depend on upstream connection

---

# 14. Malware Behavior Without Internet

Malware fails to execute:

* cannot reach C2
* cannot verify environment
* may not encrypt files

---

## Why?

Malware checks:

* internet access
* sandbox detection

---

# 15. What You Achieved

✅ Removed attacker control
✅ Prevented reinfection
✅ Stopped external communication

---

# 16. Remaining Threat

Now only:

## ⚠️ Automated Malware Actions

Examples:

* scheduled tasks
* persistence scripts
* delayed execution
* self-replication

---

## Why Important

These do NOT need attacker connection.

---

# 17. Full Eradication Checklist

Before moving to recovery:

### Network

* [ ] C2 blocked
* [ ] malicious domains blocked

### Hosts

* [ ] infected machines identified
* [ ] malware files located

### Persistence

* [ ] web shells identified
* [ ] automated actions found

### Files

* [ ] all malware copies identified

---

# 18. Tools Summary

| Tool           | Purpose                       |
| -------------- | ----------------------------- |
| tcpdump        | analyze network traffic       |
| nmap           | find open ports (web shells)  |
| strings        | extract malware signatures    |
| YARA           | detect malware via signatures |
| Velociraptor   | scan endpoints at scale       |
| nslookup       | resolve domains               |
| firewall / ACL | block attacker traffic        |

---

# 🔥 Final Key Takeaways

* Eradication is **NOT just deleting malware**
* It requires:

  * full visibility
  * coordinated blocking
  * simultaneous action

---

## Most Important Concepts

1. **Scope EVERYTHING first**
2. **Block attacker at network level**
3. **Find malware using signatures (not names)**
4. **Use endpoint tools for full visibility**
5. **Act all at once (no partial fixes)**

---


---

# Network Analysis Incident Response – Full Notes

---

## Main Tools Used Throughout the Course

* Kibana
* Elasticsearch
* Zeek
* tcpdump

---

# 1. Investigation Workflow (Overall Strategy)

```
1 Identify suspicious communication
2 Find infected host
3 Track lateral movement
4 Identify attacker infrastructure (C2)
5 Detect enumeration / scanning
6 Investigate critical servers
7 Detect data exfiltration
8 Collect IOCs and mitigation artifacts
```

### Goal

```
Understand the full attack timeline end-to-end
```

---

# 2. Tools Explained

---

## Kibana

**Purpose**

```
Search and analyze network logs quickly
```

**Why Used**

* Query massive datasets fast
* Filter by IP, port, protocol
* Identify traffic patterns

**Example Filters**

```
source.ip = 172.31.37.10
destination.ip = 172.31.101.111
destination.port = 443
```

**What You Learn**

```
Who is communicating
Frequency of communication
Ports and protocols used
```

---

## Elasticsearch

**Purpose**

```
Stores all network logs
```

**Key Idea**

```
Kibana = visualization layer
Elasticsearch = data storage
```

---

## Zeek

**Purpose**

```
Convert packet captures into structured logs
```

**Key Logs**

```
conn.log   → connections
http.log   → web traffic
dns.log    → DNS queries
ssl.log    → TLS traffic
files.log  → transferred files
```

**Why Important**

```
Turns raw packets into readable, searchable data
```

---

## tcpdump

**Purpose**

```
Inspect raw packet-level traffic
```

**Example**

```
tcpdump -nn -r incident.pcap host 172.31.37.20
```

**Why Used**

```
Deep inspection when logs aren’t enough
```

---

# 3. Step 1 – Identify Suspicious Communication

**Goal**

```
Find unusual outbound traffic
```

**Example**

```
172.31.37.20 → iamironcat.com
```

**Indicators**

* Unknown domains
* Strange ports
* High traffic volume

---

# 4. Step 2 – Identify Infected Host

**Method**

```
Filter by source IP in Kibana
```

**Example**

```
172.31.37.20
```

**Find**

```
External connections
Communication patterns
Possible malware activity
```

---

# 5. Step 3 – Track Lateral Movement

**Question**

```
Did the attacker spread internally?
```

**Example**

```
172.31.37.10 → 172.31.37.20 (port 445)
```

**Port 445 = SMB**

**Why Important**

```
Used for file sharing and remote execution
Common for ransomware spread
```

---

# 6. SMB Investigation

**Command**

```
tcpdump -nn -r incident.pcap host 172.31.37.20
```

**Look For**

* SMB sessions
* File transfers
* Authentication attempts

---

# 7. Command & Control (C2)

**IOC Found**

```
172.31.101.111
```

**Filter**

```
source.ip = 172.31.37.10 AND destination.ip = 172.31.101.111
```

**Result**

```
11,000+ connections
```

**Meaning**

```
Active C2 communication over HTTPS (port 443)
```

---

# 8. Beaconing Detection

**Pattern**

```
Regular repeated connections to same IP
```

**Indicators**

* Consistent timing
* Similar packet sizes
* High frequency

**Meaning**

```
Malware calling home
```

---

# 9. Malware Download Detection

**Using Zeek**

```
cat http.log
```

**Look For**

* Suspicious URIs
* Downloaded files

**Example**

```
payload.bat
```

---

# 10. Detect Enumeration / Scanning

**Observation**

```
172.31.37.10 scanning network
```

**Pattern**

```
~256 attempts per port
~1900 attempts per host
```

**Type**

```
SYN scan
```

---

## SYN Scan Explained

**Normal**

```
SYN → SYN-ACK → ACK
```

**Scan**

```
SYN → SYN-ACK → (no ACK)
```

**Purpose**

```
Find open ports stealthily
```

---

# 11. Confirm Lateral Movement

After scanning:

```
Connections to SMB (445)
```

**Meaning**

```
Ransomware spreading across hosts
```

---

# 12. Investigate Critical Server

**Target**

```
172.31.64.10
```

**Role**

```
Critical application server
```

---

# 13. Suspicious DNS Activity

**Filter**

```
source.ip = 172.31.64.10
```

**Observation**

```
Large volume of DNS requests
```

**Destination**

```
172.31.0.2 (internal DNS)
```

**Insight**

```
Direction normal, volume abnormal
```

---

# 14. DNS Entropy Analysis

**Tool**

```
Zeek + dns_entropy script
```

**Purpose**

```
Detect encoded/random DNS queries
```

---

## Entropy Concept

**Normal**

```
google.com → low entropy (~2.5)
```

**Suspicious**

```
a8sd9f8sd9f.domain.com → high entropy
```

**Meaning**

```
Encoded data inside DNS queries
```

---

# 15. DNS Exfiltration Discovery

**Domain**

```
dirtylitter.iamironcat.com
```

**Pattern**

```
encodeddata.dirtylitter.iamironcat.com
```

**Technique**

```
DNS tunneling
```

---

## Key Commands

**Count Queries**

```
grep dirtylitter dns_entropy.log | wc -l
```

Result:

```
4260 queries
```

---

**Find Source Hosts**

```
cut -f1 dns_entropy.log | sort | uniq
```

Result:

```
172.31.64.10 only
```

---

# 16. True Objective of Attack

**Initial Assumption**

```
Ransomware
```

**Reality**

```
Data exfiltration
```

**Insight**

```
Ransomware = distraction
```

---

# 17. Full Attack Timeline

```
1 Phishing email delivered
2 Malicious macro executed
3 Malware contacts C2
4 Payload downloaded
5 Internal SYN scan
6 SMB hosts discovered
7 Ransomware deployed
8 Critical server accessed
9 Data exfiltrated via DNS
```

---

# 18. Indicators of Compromise (IOCs)

**IP**

```
172.31.101.111
```

**Domain**

```
dirtylitter.iamironcat.com
```

**File**

```
payload.bat
```

**Ports**

```
443 (C2)
8000 (web shell)
445 (SMB)
53 (DNS)
```

**Techniques**

```
Beaconing
SYN scanning
SMB lateral movement
DNS tunneling
```

---

# 19. Key Lessons

1. Always analyze **internal (east-west) traffic**
2. Combine:

   ```
   Host analysis + Network analysis
   ```
3. Ransomware may hide **secondary objectives**
4. Watch for:

   ```
   Beaconing
   Scanning
   Lateral movement
   Exfiltration
   ```
5. DNS can be used for **covert data transfer**

---

# 20. Investigation Mindset

Always ask:

```
Who talked to who?
How often?
Why that port?
Is this normal behavior?
What happened before?
What happened after?
```

---



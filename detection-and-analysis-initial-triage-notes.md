# Incident Response Initial Triage – 

This guide shows **how to quickly collect critical evidence from a compromised system** before it changes or disappears.

Goal of this phase:

* Preserve volatile evidence
* Identify attacker behavior
* Gather indicators of compromise (IOCs)
* Begin scoping other infected systems

---

# 1. Create a Folder to Store Evidence

First create a directory to store everything you collect from the system.

### Command

```powershell
$name = $env:computername
$domain = $env:USERDOMAIN
$datapath = "$name-$domain"
new-item -Name $datapath -Path . -ItemType directory
```

### What it does

* Gets the **computer name**
* Gets the **domain name**
* Creates a folder named like:

```
COMPUTERNAME-DOMAIN
```

### Why this matters

You may analyze **multiple machines**, so each system needs its own evidence folder.

---

# 2. Create an Activity Log

Always document every action taken on a system.

### Command

```powershell
function captains_log($entry, $datapath){
    $datetime = Get-Date
    $timezone = (Get-TimeZone).DisplayName
    $stardate = "Log Entry: $datetime $timezone"
    $stardate | out-file -Append -FilePath ./$datapath/master-station-log.txt
    $entry | out-file -Append -FilePath ./$datapath/master-station-log.txt
}
```

### Example use

```powershell
captains_log "Started initial triage." $datapath
```

### What it does

Creates a log file called:

```
master-station-log.txt
```

Each entry records:

* date
* time
* timezone
* analyst note

### Why this matters

Incident response requires **chain-of-custody and documentation**.

---

# 3. Record Basic System Information

Collect fundamental system data immediately.

### Commands

```powershell
get-date >> ./$datapath/master-station-log.txt
Get-TimeZone >> ./$datapath/master-station-log.txt
Get-NetIPInterface >> $datapath/master-station-log.txt
Get-NetIPAddress >> $datapath/master-station-log.txt
```

### What they do

| Command            | Purpose                   |
| ------------------ | ------------------------- |
| get-date           | records system date/time  |
| Get-TimeZone       | records timezone          |
| Get-NetIPInterface | lists network adapters    |
| Get-NetIPAddress   | shows system IP addresses |

### Why this matters

Helps identify:

* which system you are on
* network addresses used
* timeline correlation later

---

# 4. Record All Terminal Activity

Start logging everything typed in the terminal.

### Command

```powershell
start-transcript -Path ./$datapath/powershelltranscript.txt -Append
```

### What it does

Records **all commands and output** to:

```
powershelltranscript.txt
```

### Alternative (GUI recorder)

```powershell
psr
```

This launches **Problem Steps Recorder**, which records:

* screenshots
* clicks
* typed commands

### Why this matters

Ensures **nothing during investigation is lost**.

---

# 5. Collect Detailed System Information

### Command

```powershell
get-computerinfo
```

### What it does

Collects:

* OS version
* hardware
* patches
* system configuration

### Why this matters

Helps identify:

* vulnerable OS
* patch level
* system role

---

# 6. Identify System Architecture

Determine if system is **32-bit or 64-bit**.

### Command

```powershell
wmic OS get OSArchitecture
```

### Why this matters

You must run **tools compiled for the correct architecture**.

Example:

* 32-bit tools for 32-bit OS
* 64-bit tools for 64-bit OS

---

# 7. Dump System Memory (VERY IMPORTANT)

Memory contains **the most volatile evidence**.

### Command

```powershell
./winpmem_mini_x64_rc2.exe $datapath/$computername-memorydump.dmp
```

### What it does

Creates a **RAM memory dump file**.

### Why this matters

Memory may contain:

* malware code
* encryption keys
* command-and-control connections
* attacker tools

Later analysis tools:

* Volatility
* Rekall

---

# 8. Document Suspicious Files (Example: Ransom Note)

### Get file details

```powershell
$note_file_details = get-itemproperty -Path C:\Users\Administrator\Desktop\ransomnote.txt
$note_file_details | out-file -Filepath $datapath/IOC-ransom-note.txt -append
```

### What it does

Collects:

* file creation time
* modification time
* file size
* metadata

### Why this matters

The timestamp often reveals **when the attack started**.

---

# 9. Generate File Hash (Indicator of Compromise)

### Command

```powershell
$filehash = get-filehash -Algorithm md5 -Path C:\Users\Administrator\Desktop\ransomnote.txt
$filehash | out-file -Filepath $datapath/IOC-ransom-note.txt -append
```

### What it does

Creates a **file fingerprint**.

### Why this matters

The hash can be used to:

* detect the same malware elsewhere
* build detection rules
* share threat intelligence

---

# 10. Copy Suspicious Files for Analysis

### Command

```powershell
copy-item -Path C:\Users\Administrator\Desktop\ransomnote.txt $datapath
```

### What it does

Copies the suspicious file into the evidence folder.

### Why this matters

Preserves the file for:

* malware analysis
* reverse engineering

---

# 11. Search Entire System for Similar Files

### Command

```powershell
get-childitem c:\ -filter *ransomnote* -recurse
```

### What it does

Searches the entire file system for files matching the pattern.

### Why this matters

Helps determine:

* which folders were encrypted
* how widespread the attack is

---

# 12. Check Windows Event Logs

### Command

```powershell
get-winevent
```

### What it does

Displays Windows event logs.

### What to look for

Particularly important event ID:

```
1102
```

Meaning:

```
Audit log cleared
```

### Why this matters

Attackers often **delete logs to hide activity**.

---

# 13. Check Volume Shadow Copies

Ransomware often deletes backups.

### Command

```powershell
vssadmin list shadows
```

### What it does

Lists existing **Windows shadow backups**.

### Why this matters

If backups exist:

* files might be recoverable

If deleted:

* attacker attempted to block recovery

---

# 14. Inspect Active Network Connections

### Command

```powershell
Get-NetTCPConnection
```

### What it does

Shows:

* active TCP connections
* listening ports

### Why this matters

Helps identify:

* command and control servers
* malware communication

---

# 15. Inspect Network Connections (Detailed)

### Command

```powershell
netstat -anob
```

### What it does

Displays:

* open ports
* active connections
* associated processes

### Why this matters

Links **network activity to a process**.

---

# 16. Save Network Data

### Commands

```powershell
$netConn = get-nettcpconnection
$netconn | export-clixml -Path $datapath/netconn.xml
$netconn | out-file -FilePath $datapath/netconn.txt
```

### What it does

Saves network connection data.

### Why this matters

Allows **later analysis without needing the system again**.

---

# 17. Use Sysinternals to Inspect Connections

### Command

```powershell
./SysInternalsSuite/tcpvcon64.exe -a /accepteula
```

### What it does

Shows:

* TCP connections
* process IDs
* executable names

### Why this matters

Provides **more detail than netstat**.

---

# 18. List Running Processes

### Command

```powershell
$procs = Get-Process
$procs
```

### What it does

Displays all running processes.

### Why this matters

Malware runs as a **process**.

Look for:

* unusual names
* high CPU
* unknown executables

---

# 19. Save Process List

### Command

```powershell
$procs |out-file -path $datapath/processes.txt
```

### Why this matters

Preserves process state for later investigation.

---

# 20. Get Detailed Process Information

### Command

```powershell
wmic process list full /format:htable >> $datapath/process.html
```

### What it does

Creates a detailed **HTML report** containing:

* command line arguments
* executable path
* parent processes

### Why this matters

Helps identify **how malware was launched**.

---

# 21. Investigate Processes with Process Explorer

### Command

```powershell
./SysInternalsSuite/procexp64.exe -accepteula
```

### What it does

Launches **Process Explorer GUI**.

### Why this matters

Allows you to inspect:

* parent-child processes
* loaded DLLs
* network activity

---

# 22. Inspect Network Connections Graphically

### Command

```powershell
./SysInternalsSuite/tcpview64.exe
```

### What it does

Shows **live TCP/UDP connections**.

### Why this matters

Easier to detect suspicious connections visually.

---

# 23. Stop Command Recording

Once finished collecting data:

### Command

```powershell
stop-transcript
```

---

# 24. Identify Other Infected Systems

### Get local network devices

```powershell
arp
```

### What it does

Shows devices communicating with this system.

---

### Test suspicious port on other machines

```powershell
Test-NetConnection <IP> -Port <PORT>
```

### Example

```powershell
Test-NetConnection 192.168.1.20 -Port 8080
```

### Why this matters

If the same port is open on other machines, they may also be compromised.

---

# 25. Collect Additional System Evidence

### DNS Cache

```powershell
$dnsCache = Get-DnsClientCache
$dnsCache | Export-Clixml -Path $datapath/dnscache.xml
$dnsCache | out-file $datapath/dnscache.txt
```

Shows recently contacted domains.

---

### ARP Cache

```powershell
$arpCache  = Get-NetNeighbor
$arpCache | Export-Clixml -Path $datapath/arpcache.xml
$arpCache | out-file $datapath/arpcache.txt
```

Shows nearby devices.

---

### Routing Table

```powershell
$routeTable = Get-NetRoute
$routeTable | Export-Clixml -Path $datapath/routetable.xml
$routeTable | out-file $datapath/routetable.txt
```

Shows network routes.

---

# 26. Collect User Information

### Commands

```powershell
net user
lusrmgr
net local group administrators
net group administrators
```

### What it does

Shows:

* local users
* admin accounts

### Why this matters

Attackers often create **new admin accounts**.

---

# 27. Identify Logged-in Users

### Commands

```powershell
./SysInternalsSuite/psloggedon64.exe -accepteula
./SysInternalsSuite/loggedonsessions.exe -accepteula
```

Shows active sessions.

---

# 28. Identify Services (Persistence)

### Commands

```powershell
sc query
wmic service list config
```

### Why this matters

Malware often installs **malicious services**.

---

# 29. Check Startup Persistence

### Command

```powershell
./SysInternalsSuite/autoruns64.exe
```

### What it does

Shows all programs configured to start automatically.

---

# 30. Collect Event Logs

### Command

```powershell
wevutil qe security /f:text
```

### Copy logs for analysis

```powershell
new-item -type directory winevent_logs

copy-item -Recurse -path C:\Windows\System32\Winevt\Logs\ -Destination ./winevent_logs

copy-item -recurse -path C:\Windows\System32\LogFiles\ -Destination ./winevent_logs
```

---

# 31. Capture Network Traffic

### Command

```powershell
rawpcap.exe $datapath/$computername-init-pcap.pcap
```

### What it does

Captures **live network packets**.

### Why this matters

Can reveal:

* malware communications
* attacker commands

---

# 32. Analyze Network Traffic (Later)

Tools used later:

```
suricata
zeek
zeekcut
```

# 33. Quick Threat Intelligence Checks (IOC Enrichment)

After collecting indicators such as:

* suspicious **domains**
* **IP addresses**
* **encoded strings**
* **file hashes**

You should perform **quick threat intelligence checks** to see if anyone else has seen them before.

⚠️ Important:
This step should take **only a few minutes**. Incident responders should **not spend hours doing threat intel research** — that is a separate role.

Your goal is simply to determine:

* Has this threat been seen before?
* Are there known behaviors or tools associated with it?

---

# 34. Decode Suspicious Encoded Strings (Base64)

Attackers often encode commands or messages using **Base64 encoding**.

Example indicators might look like:

```
QmFzZTY0IGVuY29kZWQgc3RyaW5n
```

This is not encryption — just **encoding**, and it is easy to decode.

---

## Tool: CyberChef

Use the tool:

```
https://gchq.github.io/CyberChef
```

CyberChef is a trusted tool developed by the UK intelligence agency **GCHQ**.

⚠️ Avoid other fake copies of this website — some are malicious.

---

## Steps to Decode Base64

1. Open **CyberChef**
2. Paste the suspicious string into the **input field**
3. Add the operation:

```
From Base64
```

CyberChef will automatically decode the string.

---

### Why this matters

Decoded strings sometimes reveal:

* attacker commands
* URLs or IPs
* malware configuration
* encryption keys
* attacker messages

Sometimes the decoded text may only contain **taunts or junk data**, but checking takes only seconds.

---

## Example

Encoded string:

```
SGVsbG8gd29ybGQ=
```

Decoded result:

```
Hello world
```

---

# 35. Verify if a String is Actually Base64

Not every strange string is Base64.

If decoding produces **garbage output**, the string may be:

* random data
* an encryption key
* another encoding method
* a unique victim ID

Example:

```
Windows_xxxxx_xxxxx
```

If removing clear-text parts and decoding still fails, it likely **is not Base64**.

---

# 36. Check Suspicious Domains in URL Intelligence Databases

If malware contacts a domain, check whether it is already known to be malicious.

---

## Tool: URLHaus

Website:

```
https://urlhaus.abuse.ch
```

URLHaus is a public database of **malicious command-and-control URLs**.

---

### Steps

1. Open URLHaus
2. Go to the **Database**
3. Search the suspicious domain.

Example search:

```
suspiciousdomain.com
```

---

### Possible results

| Result        | Meaning                          |
| ------------- | -------------------------------- |
| Domain listed | Known malware infrastructure     |
| No results    | May be new or previously unknown |

Even **no result is useful information** — it may indicate a **new campaign**.

---

# 37. Check Indicators in VirusTotal

Another fast threat-intel check is using **VirusTotal**.

Tool:

VirusTotal

Website:

```
https://www.virustotal.com
```

VirusTotal aggregates results from **70+ antivirus engines and threat intel feeds**.

---

## What you can search

VirusTotal supports searching:

* domains
* IP addresses
* URLs
* file hashes
* files

---

### Example Domain Search

Search:

```
suspiciousdomain.com
```

Possible outcomes:

| Result              | Meaning                        |
| ------------------- | ------------------------------ |
| Detected by engines | Known malicious infrastructure |
| Clean result        | Possibly new threat            |

---

# 38. Use VirusTotal Graph to Discover Related Infrastructure

One powerful feature of VirusTotal is the **Graph view**.

This shows relationships between:

* domains
* IP addresses
* files
* malware samples

---

### Steps

1. Sign in to VirusTotal (free account)
2. Search the domain
3. Open the **Graph view**
4. Expand the node to see related objects.

This can reveal:

* malware samples contacting the domain
* other related domains
* infrastructure used by the attacker

---

# 39. Identify Related Malware Samples

Sometimes VirusTotal will show **malware samples that communicate with the domain**.

Example detection:

```
5 / 67 engines detect malware
```

Even a few detections can indicate suspicious behavior.

---

### Look for behavioral analysis results

Example findings:

* file deletion activity
* command line execution
* persistence mechanisms

Example behavior:

```
File deletion via command line
```

---

### Why this matters

If your malware matches the behavior of the sample, you now know to investigate:

* suspicious file deletion
* self-deleting malware
* temporary file creation

---

# 40. Examine Domain Information (WHOIS)

Domain registration information can reveal useful clues.

Information includes:

* domain registrar
* domain creation date
* DNS provider
* ownership privacy settings

Example findings:

```
Registrar: GoDaddy
DNS Provider: Microsoft Azure
```

---

### Why this matters

Threat actors often reuse:

* domain registrars
* hosting providers
* infrastructure patterns

These patterns can help link attacks together.

---

# 41. Identify Hosting Infrastructure

Threat infrastructure often uses large cloud providers such as:

* Microsoft Azure
* Amazon AWS
* Google Cloud

This does **not mean the cloud provider is malicious** — attackers simply use them as hosting platforms.

---

# 42. Check Domain Subdomains

A domain may have multiple **subdomains**.

Example:

```
domain.com
hello.domain.com
api.domain.com
update.domain.com
```

Subdomains may be used for:

* command and control
* payload delivery
* phishing portals

Threat intel tools may show **related subdomains automatically**.

---

# 43. Visit Suspicious Domains Carefully

Sometimes you may manually inspect the website.

⚠️ Only do this in a **secure sandbox environment**.

Possible observations:

* ransomware payment portals
* command panels
* phishing pages

Example ransomware portal behavior:

* asks for victim ID
* asks for email
* provides cryptocurrency payment address

---

# 44. Collect Additional Indicators

While examining attacker infrastructure, record new indicators such as:

* payment cryptocurrency addresses
* victim identifiers
* additional domains
* additional IP addresses

Example IOC types:

```
Domain
IP address
File hash
Process name
Cryptocurrency wallet
Network port
```

These indicators can later be used to:

* detect additional infections
* block malicious traffic
* build detection rules

---

# 45. Know When to Stop Threat Intel Research

Incident responders should **not spend too long on intelligence research**.

Once basic context is gathered:

✔ stop analysis
✔ return to incident investigation

Deep intelligence analysis should be handled by:

* threat intelligence teams
* malware analysts
* digital forensics specialists

---



# Host Data Collection (Live Response)

Once initial triage is complete and a session is established on the victim host, the next step is **host data collection**. At this stage we are **collecting artifacts only**—not performing full analysis. The purpose is to preserve volatile and system information so that it can be analyzed later in a controlled environment.

All collected data should be stored in the **same data path used during initial triage**. This ensures artifacts are organized and easy to review later.

---

# Start the PowerShell Transcript

Before collecting artifacts, start a **PowerShell transcript**. This records every command executed and its output, providing a complete record of the response activity.

```powershell
Start-Transcript -Path $datapath\host_collection_transcript.txt
```

It is also useful to add a **timestamped log entry** for documentation purposes.

```powershell
echo "captains_log: Host data collection started $(Get-Date)" >> $datapath\captains_log.txt
```

This provides a time-stamped entry documenting the beginning of host collection.

---

# System Information

One of the first pieces of data to collect is **general system information**.

In PowerShell this can be done with:

```powershell
Get-ComputerInfo
```

Alternatively, the traditional command-line tool can be used:

```powershell
systeminfo
```

This collects details such as:

* OS version
* patch level
* system architecture
* installed hotfixes
* hardware information

The output will automatically be captured inside the PowerShell transcript.

---

# DNS Cache

The **DNS cache** provides insight into domains that were recently resolved by the system.

This is extremely valuable during incident response because it can reveal:

* command-and-control domains
* malicious infrastructure
* recently contacted services

```powershell
Get-DnsClientCache
```

Example output may show entries such as:

```
hello.iamironcat -> IP address
```

This allows investigators to determine what domains were recently resolved and where they pointed.

Save the results:

```powershell
Get-DnsClientCache | Out-File $datapath\dnsCache.txt
```

---

# ARP Cache

The **ARP cache** contains mappings of IP addresses to MAC addresses on the local subnet.

This helps identify **other devices the host recently communicated with**.

PowerShell command:

```powershell
Get-NetNeighbor
```

However, this command may not exist on older systems (such as Windows Server 2012 or Windows 7).

In those cases, use:

```powershell
arp -a
```

Save the results:

```powershell
arp -a > $datapath\arp.txt
```

This ensures network neighbor information is preserved for later investigation.

---

# Routing Table

Routing tables can contain **temporary (ephemeral) routes** created during the current OS session.

Malware sometimes modifies routes to redirect or intercept traffic.

Collect the routing table:

```powershell
Get-NetRoute
```

Save the output:

```powershell
Get-NetRoute | Out-File $datapath\routes.txt
```

---

# Firewall Rules

Local firewall rules may be manipulated by attackers to:

* allow inbound command-and-control connections
* enable lateral movement
* create proxy tunnels

Collect firewall rules with:

```powershell
Get-NetFirewallRule | Out-File $datapath\firewallrules.txt
```

The output is typically very large, which is why it is stored for **later analysis rather than immediate review**.

---

# Local Users and Groups

Investigators should verify whether **unauthorized accounts were created**.

List local users:

```powershell
net user
```

List local administrator group members:

```powershell
net localgroup administrators
```

These commands help identify:

* suspicious local accounts
* privilege escalation attempts

---

# Logged-On Users

To determine who is currently logged into the system:

```powershell
psloggedon
```

For more detailed session information:

```powershell
logonsessions64
```

This tool provides:

* session IDs
* authentication package
* logon timestamps

Session IDs are especially useful because they appear frequently in **Windows security logs**.

---

# Domain User Enumeration (Optional)

If the system is domain joined and **Active Directory tools are installed**, domain users can be enumerated:

```powershell
Get-ADUser -Filter *
```

This retrieves all users within the domain.

---

# Shared Resources

Attackers and ransomware frequently abuse **network shares** for lateral movement.

List active shares:

```powershell
net use
```

If shares exist, they should be investigated further.

---

# Service Enumeration

Malware commonly installs itself as a **Windows service** to maintain persistence.

However, in PowerShell the `sc` command conflicts with a PowerShell alias.

To run the real command-line tool:

```powershell
cmd /c sc query
```

This lists:

* service names
* current status
* running state

Save results:

```powershell
cmd /c sc query > $datapath\services.txt
```

Another method is via WMI:

```powershell
Get-WmiObject Win32_Service
```

This provides additional service metadata.

---

# Persistence Locations (Autoruns)

A powerful tool for identifying persistence mechanisms is **Autoruns** from Sysinternals.

Autoruns displays:

* services
* drivers
* scheduled tasks
* startup entries
* registry autoruns
* print monitors

Run Autoruns and save the configuration snapshot:

```
autoruns.exe
```

Then export the results to the collection directory.

This snapshot can later be opened with the same Autoruns tool for detailed analysis.

---

# Scheduled Tasks

Scheduled tasks are another common persistence method.

Collect them using:

```powershell
schtasks /query /fo LIST /v > $datapath\schtasks.txt
```

---

# Disk and Volume Information

While running on the live system, a **full disk image cannot be created**, because the OS is currently using the disk.

However, we can collect filesystem metadata.

Example:

```powershell
volumeid
```

NTFS metadata collection:

```powershell
ntfsinfo C:
```

This retrieves:

* MFT information
* filesystem metadata
* NTFS configuration

---

# Disk Activity Monitoring

Disk activity can also be monitored temporarily using **DiskMon** (Sysinternals).

This tool tracks:

* file reads
* file writes
* disk access patterns

Captured activity can then be saved for later review.

---

# DLL and Process Module Information

Processes load **dynamic-link libraries (DLLs)** known as modules.

Listing DLLs can reveal injected libraries or malicious modules.

Example:

```powershell
listdlls
```

Process handles can also reveal detailed resource access by malware.

---

# Security Logs

Windows logs are critical for incident reconstruction.

Security logs can be exported using the command line:

```powershell
wevtutil qe Security
```

Alternatively, export logs directly to a file.

---

# Copy All Event Logs

Rather than collecting logs individually, responders often copy the entire **Windows Event Log directory**.

Create a directory for logs:

```powershell
mkdir $datapath\winevent_logs
```

Copy logs:

```powershell
Copy-Item C:\Windows\System32\winevt\Logs\* $datapath\winevent_logs -Recurse
```

These `.evtx` files can later be opened in **Event Viewer or forensic tools**.

---

# Copy Additional System Logs

Some logs are stored outside the Windows Event system.

A valuable location is:

```
C:\Windows\System32\LogFiles
```

Copy them as well:

```powershell
Copy-Item C:\Windows\System32\LogFiles\* $datapath\LogFiles -Recurse
```

Some errors may occur due to permissions. This is normal.

To suppress errors:

```powershell
-ErrorAction Continue
```

---

# Firewall Log Files

Inside the copied log files, the firewall logs are particularly valuable:

```
LogFiles\Firewall
```

These logs record:

* allowed connections
* blocked connections
* source/destination IPs
* timestamps

This data can be used to reconstruct network activity and track lateral movement.

---

# Next Step: Full Disk Imaging

After host collection is complete, the next step is to obtain a **full disk image**.

A disk image enables deep forensic analysis including:

* registry artifacts
* ShimCache
* AmCache
* NTFS metadata
* deleted files
* timeline reconstruction

Disk imaging is typically performed **offline**, often by booting a forensic OS (such as Linux) and imaging the drive.

---


# Host Artifact Collection (Live Response)

After completing **initial triage**, the next step is **host data collection**.

⚠️ At this stage:

* We **collect artifacts only**
* We **do not perform deep analysis yet**
* All collected data will be analyzed **later in a forensic environment**

All artifacts should be saved to the **same data collection directory created during triage**, typically referenced by the variable:

```powershell
$datapath
```

This ensures all evidence remains **organized and centralized**.

---

# 1. Start a PowerShell Transcript

Before collecting artifacts, start a **PowerShell transcript** to record all commands and outputs.

```powershell
Start-Transcript -Path $datapath\host_collection_transcript.txt
```

This provides a **complete audit trail** of actions performed during incident response.

Add a timestamp entry to the responder log:

```powershell
echo "captains_log: Host collection started $(Get-Date)" >> $datapath\captains_log.txt
```

This records the **time and timezone of the collection process**.

---

# 2. Collect System Information

Basic system information provides context about the host.

PowerShell method:

```powershell
Get-ComputerInfo
```

Command line alternative:

```powershell
systeminfo
```

This collects information such as:

* OS version
* system architecture
* installed patches
* host name
* domain membership
* hardware details

The output is automatically captured inside the transcript.

---

# 3. Collect DNS Cache

The **DNS cache** contains recently resolved domains.

This can reveal:

* command-and-control domains
* malicious infrastructure
* recently accessed websites

Collect DNS cache:

```powershell
Get-DnsClientCache | Out-File $datapath\dnsCache.txt
```

Example finding:

```
hello.iamironcat → IP address
```

This shows the domain that the host recently resolved and the IP address it mapped to.

---

# 4. Collect ARP Cache

The **ARP cache** shows recent network neighbors on the local subnet.

Preferred PowerShell command:

```powershell
Get-NetNeighbor
```

However, this command may not exist on older systems.

Fallback method:

```powershell
arp -a
```

Save results:

```powershell
arp -a > $datapath\arp.txt
```

This preserves network device relationships for later analysis.

---

# 5. Collect Routing Table

Routing tables may contain **temporary (ephemeral) routes** created during the current session.

Malware may manipulate routes to redirect traffic.

Collect routing table:

```powershell
Get-NetRoute | Out-File $datapath\routes.txt
```

Routes may include:

* static routes
* dynamic routes
* temporary routes created during runtime

Capturing them preserves potentially malicious network changes.

---

# 6. Collect Firewall Rules

Local firewall rules may be modified by attackers to:

* allow command-and-control traffic
* enable lateral movement
* create proxy paths through the system

Collect firewall rules:

```powershell
Get-NetFirewallRule | Out-File $datapath\firewallrules.txt
```

This output is typically large and is reviewed later during analysis.

---

# 7. Enumerate Local Users and Groups

Attackers often create **new accounts or modify privileges**.

List local users:

```powershell
net user
```

List administrator group members:

```powershell
net localgroup administrators
```

Investigators should check for:

* unknown accounts
* recently created users
* privilege escalation activity

Even domain-joined systems still maintain **local accounts**.

---

# 8. Identify Logged-On Users

Identify users currently logged into the system.

Using Sysinternals tool:

```powershell
psloggedon
```

More detailed session information:

```powershell
logonsessions64
```

This provides:

* session ID
* authentication package
* logon timestamp

Session IDs are often referenced in **Windows security logs**.

---

# 9. Enumerate Domain Users (If Domain Joined)

If Active Directory tools are available, enumerate domain users:

```powershell
Get-ADUser -Filter *
```

This retrieves all domain accounts.

---

# 10. Check Network Shares

Ransomware and malware often abuse **network shares**.

Check for shares:

```powershell
net use
```

Attackers may use shares for:

* lateral movement
* encrypting shared storage
* staging malware

---

# 11. Enumerate Services

Services are a **common persistence mechanism**.

Command prompt method:

```powershell
cmd /c sc query
```

This lists:

* service names
* running status
* service state

Save results:

```powershell
cmd /c sc query > $datapath\services.txt
```

PowerShell alternative:

```powershell
Get-WmiObject Win32_Service
```

Different tools may reveal **different metadata**, which is why multiple methods are useful.

---

# 12. Identify Persistence with Autoruns

The **Autoruns tool** identifies persistence mechanisms across many locations.

Run:

```
autoruns.exe
```

Autoruns displays:

* startup entries
* services
* drivers
* scheduled tasks
* registry run keys
* browser extensions

Export the Autoruns snapshot to the collection directory for later analysis.

---

# 13. Collect Scheduled Tasks

Scheduled tasks are frequently used for persistence.

Collect tasks:

```powershell
schtasks /query /fo LIST /v > $datapath\schtasks.txt
```

This records:

* task names
* execution times
* run commands
* user context

---

# 14. Collect Disk and Filesystem Information

While the system is live, it is not possible to create a full disk image of the running OS.

However, filesystem metadata can still be collected.

Example:

```powershell
volumeid
```

Collect NTFS information:

```powershell
ntfsinfo C:
```

This reveals:

* NTFS configuration
* Master File Table information
* filesystem metadata

---

# 15. Monitor Disk Activity (Optional)

Disk activity monitoring can reveal suspicious file access.

Tool:

```
diskmon
```

This tracks:

* file reads
* file writes
* filesystem activity

Captured data can be saved for later analysis.

---

# 16. Enumerate Process Modules

Processes load **dynamic link libraries (DLLs)**.

Listing DLL modules can identify:

* injected libraries
* malicious modules

Example tool:

```powershell
listdlls
```

This shows which DLLs are loaded into each process.

---

# 17. Export Windows Event Logs

Security logs are critical for reconstructing attacker activity.

Example command:

```powershell
wevtutil qe Security
```

Alternatively, copy all event logs.

Create log directory:

```powershell
mkdir $datapath\winevent_logs
```

Copy logs:

```powershell
Copy-Item C:\Windows\System32\winevt\Logs\* $datapath\winevent_logs -Recurse
```

These `.evtx` files can later be analyzed in forensic tools.

---

# 18. Copy Additional System Log Files

Some logs are stored outside the Windows event system.

Example location:

```
C:\Windows\System32\LogFiles
```

Copy them:

```powershell
Copy-Item C:\Windows\System32\LogFiles\* $datapath\LogFiles -Recurse -ErrorAction Continue
```

Errors may occur due to permissions, which is normal.

---

# 19. Review Firewall Logs

Inside the collected logs, firewall logs are particularly useful.

Example location:

```
LogFiles\Firewall
```

Firewall logs show:

* allowed connections
* blocked connections
* source and destination IP addresses
* timestamps

These logs can help reconstruct **network activity and lateral movement**.

---

# 20. Next Step: Full Disk Imaging

Once live collection is complete, the next step is **full disk imaging**.

This is typically performed by:

* booting a forensic operating system
* imaging the disk offline

Full disk images allow deeper analysis including:

* registry artifacts
* ShimCache
* AmCache
* deleted files
* full NTFS metadata
* forensic timeline reconstruction

---

# Host & Network Data Collection Guide (Incident Response Triage)

## 1. Objective

The purpose of this phase is **data collection only**. No analysis is performed on the affected system. The goal is to gather volatile and semi-volatile artifacts so they can be analyzed later in a forensic environment.

Collected data should be stored in the **designated triage directory** (`$datapath`) created during initial triage.

---

# 2. Start Logging the Session

Before collecting artifacts, start a PowerShell transcript so every command executed during triage is recorded.

```powershell
Start-Transcript -Path $datapath\transcript.txt
```

Create a timestamped note documenting the action.

```powershell
"Captains_Log: Beginning host data collection $(Get-Date)" >> $datapath\captains_log.txt
```

This ensures all activity is **auditable and reproducible**.

---

# 3. System Information Collection

## System Configuration

Collect basic host information such as OS version, patch level, and hardware details.

PowerShell method:

```powershell
Get-ComputerInfo > $datapath\systeminfo.txt
```

Command line alternative:

```powershell
systeminfo > $datapath\systeminfo.txt
```

This provides:

* OS version
* Installed patches
* Boot time
* Network configuration
* System architecture

---

# 4. Network Artifact Collection

Network artifacts are **volatile** and should be collected early.

---

## DNS Cache

DNS cache reveals **recent domain resolutions** made by the system.

```powershell
ipconfig /displaydns > $datapath\dnsCache.txt
```

Why this matters:

* Maps **domains to IP addresses**
* Shows **recently accessed domains**
* May reveal **command-and-control infrastructure**

Example finding:

```
hello.iamironcat.com → 52.251.113.157
```

---

## ARP Cache

The ARP cache shows devices the host recently communicated with on the **local network**.

PowerShell alternative:

```powershell
Get-NetNeighbor
```

However, older systems may not support this cmdlet.

Fallback:

```powershell
arp -a > $datapath\arp.txt
```

This helps identify:

* Other hosts on the subnet
* Potential lateral movement targets

---

## Routing Table

Routes can be **static or temporary**. Malware may manipulate routing behavior.

```powershell
Get-NetRoute > $datapath\routes.txt
```

Why this matters:

* Detect **malicious route injection**
* Identify **traffic redirection**

---

# 5. Firewall Configuration

Threat actors may modify firewall rules to:

* Enable backdoors
* Allow outbound C2 traffic
* Create proxy tunnels

Collect local firewall rules:

```powershell
netsh advfirewall firewall show rule name=all > $datapath\firewallrules.txt
```

---

# 6. User and Authentication Data

## Local Users

Check all local accounts.

```powershell
net user > $datapath\local_users.txt
```

Look for:

* Unexpected accounts
* Recently created accounts
* Privileged users

---

## Local Administrators

```powershell
net localgroup administrators > $datapath\local_admins.txt
```

Important because attackers often add accounts to **Administrators** for persistence.

---

## Logged-In Users

Using Sysinternals:

```powershell
psloggedon.exe > $datapath\loggedon_users.txt
```

---

## Logon Sessions

```powershell
logonsessions64.exe > $datapath\logon_sessions.txt
```

This provides:

* Session IDs
* Authentication package
* Logon times

These session IDs are important in Windows logs and may be used in **session hijacking attacks**.

---

# 7. Network Shares

Check for mapped or shared resources.

```powershell
net use > $datapath\network_shares.txt
```

Why this matters:

Ransomware often spreads through **shared drives**.

---

# 8. Services (Persistence Check)

Malware frequently installs itself as a **Windows service**.

### Command Prompt version

Because PowerShell aliases `sc` incorrectly:

```powershell
cmd /c sc query > $datapath\services.txt
```

### WMI alternative

```powershell
Get-WmiObject Win32_Service > $datapath\services_wmi.txt
```

These reveal:

* Running services
* Disabled services
* Suspicious service names

---

# 9. Autoruns (Persistence Locations)

A powerful tool for identifying persistence mechanisms.

Use:

**Autoruns (Sysinternals)**

Check locations such as:

* Startup folders
* Services
* Drivers
* Scheduled tasks
* Registry run keys

Export results:

```
Autoruns → Save → $datapath\autoruns.arn
```

---

# 10. Scheduled Tasks

Scheduled tasks are a **common persistence technique**.

```powershell
schtasks /query /fo LIST /v > $datapath\scheduled_tasks.txt
```

---

# 11. Disk and NTFS Information

Full disk imaging cannot be performed from the live OS, but metadata can still be collected.

---

## Volume ID

```powershell
volumeid.exe > $datapath\volumeid.txt
```

---

## NTFS Metadata

```powershell
ntfsinfo.exe C: > $datapath\ntfsinfo.txt
```

This provides:

* MFT information
* NTFS metadata
* Volume timestamps

---

# 12. Process Modules (DLLs)

Processes load dynamic link libraries (DLLs) which may indicate malicious code.

```powershell
listdlls.exe > $datapath\dlls.txt
```

---

# 13. Process Handles

Handles show what resources processes are accessing.

```powershell
handle.exe > $datapath\handles.txt
```

---

# 14. Event Log Collection

Export Windows logs using command line tools.

```powershell
wevtutil qe Security > $datapath\security_logs.txt
```

---

## Copy Raw EVTX Files

Create directory:

```powershell
mkdir $datapath\winevent_logs
```

Copy logs:

```powershell
copy-item C:\Windows\System32\winevt\Logs\* $datapath\winevent_logs -Recurse
```

These `.evtx` files can be analyzed later with forensic tools.

---

# 15. Additional Log Files

Some logs exist outside the Windows Event system.

Copy system log files:

```powershell
copy-item C:\Windows\System32\LogFiles\* $datapath\LogFiles -Recurse -ErrorAction Continue
```

Example useful logs:

Firewall logs

```
C:\Windows\System32\LogFiles\Firewall\
```

These can show:

* Allowed connections
* Blocked connections
* Source and destination IPs

---

# 16. Network Traffic Capture

Capturing traffic helps identify:

* Command-and-control traffic
* Web shells
* Data exfiltration

---

## Wireshark (Analysis Example)

Capture traffic and filter for web shell traffic.

Example filter:

```
tcp.port == 8080
```

Example IP filter:

```
ip.addr == 52.251.113.157
```

This may reveal:

* Commands sent to web shells
* HTTP responses
* attacker activity

---

# 17. Portable Packet Capture (RawCap)

Wireshark is not portable, but **RawCap** can capture packets without installation.

---

## Identify Interfaces

```powershell
rawcap.exe
```

Example:

```
Interface 0 – External network adapter
```

---

## Capture Packets

```powershell
rawcap.exe 0 $datapath\network_capture.pcap
```

This creates a **PCAP file** for later analysis.

Stop capture with:

```
CTRL + C
```

The PCAP can later be analyzed using tools such as:

* Wireshark
* Zeek
* NetworkMiner

---

# 18. Next Step: Full Disk Image

Once triage collection is complete, the next step should be **forensic imaging** of the disk.

Live systems cannot image themselves. Instead:

Boot using a **forensic Linux environment** and capture the disk image.

Example tools:

* FTK Imager
* dd
* Guymager

The full disk image allows deeper forensic analysis of:

* Registry artifacts
* ShimCache
* File timelines
* MFT records

---

# Key Takeaway

This process gathers **critical forensic artifacts** including:

* Network connections
* User activity
* Services and persistence
* Event logs
* Firewall activity
* DNS history
* Network traffic






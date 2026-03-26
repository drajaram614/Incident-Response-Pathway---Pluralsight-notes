---

# **INCIDENT FORENSICS NOTES (Windows Logs + NTFS + Browser)**

---

# **1. Investigation Flow (Big Picture)**

### **Step-by-Step Workflow**

1. **Authentication Logs**
   → Who logged in & from where
2. **Process Execution Logs**
   → What attacker executed
3. **NTFS / MFT Analysis**
   → What changed on disk
4. **Browser Forensics**
   → What attacker did / intent
5. **Timeline Correlation**
   → Build full attack story

---

# **2. WINDOWS EVENT LOG ANALYSIS**

## **Tools Used**

* **Event Log Explorer**

  * GUI for analyzing Windows logs
  * Better filtering + custom columns

---

## **Key Event IDs**

| Event ID | Meaning             |
| -------- | ------------------- |
| **4624** | Successful logon    |
| **4625** | Failed logon        |
| **4688** | Process creation    |
| **4689** | Process termination |

---

## **Important Setup**

* Convert time → **UTC**
* Filter logs → avoid noise

---

## **Key Analysis Steps**

### **1. Find Logons**

* Filter:

  ```
  4624,4625
  ```

### **Look For**

* External IP addresses
* Failed logon spikes → brute force
* Successful logon from unknown IP

---

### **2. Analyze Successful Logons (4624)**

#### **Add Custom Columns**

* Username
* Logon Type

#### **Logon Types**

| Type | Meaning |
| ---- | ------- |
| 3    | Network |
| 5    | Service |
| 10   | RDP     |

---

### **Red Flags**

* Logon Type 10 (RDP) from external IP
* Unknown usernames
* Multiple IPs for same user

---

## **Process Execution (4688)**

### **Enable (Important)**

* Audit Policy → Process Tracking
* Enable **command-line logging**

---

### **What to Look For**

* Suspicious processes
* PowerShell, unknown EXEs
* Short-lived processes

---

### **Example Findings**

* Chrome launched → attacker browsing
* Missing logs → log rollover

---

---

# **3. NTFS FILE SYSTEM FORENSICS**

## **Core Concept**

NTFS stores **metadata → used to reconstruct activity**

---

## **Key Artifacts**

### **1. $MFT (Master File Table)**

* MOST IMPORTANT artifact
* Contains:

  * File names
  * Timestamps
  * File size
  * Sometimes file content

➡️ **Every file = one MFT entry**

---

### **2. $LogFile**

* Tracks:

  * File system changes
* Shows:

  * Create / modify / delete
* ❗ No timestamps

---

### **3. $Extend$UsnJrnl (USN Journal)**

* Tracks:

  * File changes WITH timestamps
* Useful for:

  * Detecting deleted files
  * Recent activity

---

### **Limitations**

* Both logs:

  * Small size
  * Only store recent activity

---

## **Resident Files**

* Files < ~700 bytes
* Stored **inside MFT**

➡️ Can recover:

* File contents even if deleted

---

---

# **4. NTFS TIMESTAMPS (MACB)**

## **Two Structures**

### **1. SI ($STANDARD_INFORMATION)**

* Frequently updated
* User-modifiable

---

### **2. FN ($FILE_NAME)**

* Less frequently updated
* Harder to tamper

---

## **MACB Explained**

| Type | Meaning          |
| ---- | ---------------- |
| M    | Modified         |
| A    | Accessed         |
| C    | Metadata changed |
| B    | Created (Born)   |

---

## **Key Differences (VERY IMPORTANT)**

| SI                   | FN                   |
| -------------------- | -------------------- |
| Changes often        | More stable          |
| Easier to manipulate | Harder to manipulate |

---

## **Forensic Insight**

➡️ SI ≠ FN
= possible **timestamp tampering**

---

---

# **5. MFT TIMELINE ANALYSIS**

## **Tools Used**

### **1. MFTECmd (CLI)**

* Parses MFT → outputs timeline

#### **Command Example**

```
MFTECmd.exe -f "path\to\MFT" --csv "output\folder" --csvf output.csv --at
```

### **What Each Option Does**

* `-f` → input MFT
* `--csv` → output directory
* `--csvf` → output filename
* `--at` → include all timestamps

---

### **2. Timeline Explorer**

* Opens CSV
* Allows:

  * Filtering
  * Sorting
  * Time analysis

---

### **3. MFTExplorer**

* GUI for MFT
* Shows:

  * File tree
  * Resident file contents

---

## **Analysis Workflow**

### **Step 1: Load MFT**

* ~164k entries

---

### **Step 2: Filter by Time**

* After system install date

---

### **Step 3: Filter by File Type**

* Focus:

  * `.exe`

---

### **Step 4: Identify Suspicious Files**

---

## **Key Findings**

### **1. Unauthorized User Activity**

* `accounting` user active
  ➡️ Should not be used

---

### **2. Firefox Installed**

* Indicates:

  * Interactive attacker activity

---

### **3. Suspicious Executables**

* Example:

  * `SocksEscort64.exe`

➡️ Likely malware

---

### **4. Wallet Files**

* `Wallet.txt`
* `Exodus_Wallet.pdf`

➡️ Found everywhere

---

### **5. Resident File Recovery**

* Extracted:

  * Crypto wallet data
  * Private key

---

## **What to Look For**

* Files in:

  * Downloads
  * Temp
  * User directories
* Unknown executables
* Repeated files
* Timeline clusters

---

---

# **6. BROWSER FORENSICS**

## **Tools Used**

### **1. BrowsingHistoryView (NirSoft)**

* Aggregates:

  * IE, Edge, Firefox

---

### **Setup**

* Set time → UTC
* Load → ALL history files

---

### **What It Shows**

* URL
* Timestamp
* Browser
* Visit type

---

## **Key Findings**

### **1. Wallet Files Accessed**

➡️ Confirms attacker opened them

---

### **2. Banking Activity**

* `regions.com`
* Others

➡️ Likely:

* Financial fraud

---

### **3. Webmail Access**

➡️ Possible:

* Data exfiltration

---

### **4. Multiple Browsers Used**

* IE, Edge, Chrome, Firefox, Opera

➡️ Evasion technique

---

---

# **7. CHROME FORENSICS (Hindsight Tool)**

## **Tool: Hindsight**

* Deep analysis of Chromium browsers

---

## **Command Example**

```
hindsight.exe -t UTC -f xlsx -b Chrome -i "profile\path" -o output --nocopy
```

---

## **Command Breakdown**

* `-t UTC` → normalize time
* `-f xlsx` → Excel output
* `-b Chrome` → browser type
* `-i` → input profile
* `-o` → output file
* `--nocopy` → faster processing

---

## **Artifacts Extracted**

* URLs
* Cookies
* Downloads
* Autofill
* Login data

---

## **Key Findings**

### **1. Heavy Activity**

* 1000+ URLs

---

### **2. Banking Focus**

* `regions.com`
* `onlinepayments.regions.com`

---

### **3. IP Check**

* `2ip.me`

➡️ Attacker verifying identity

---

### **4. Multiple Banks**

➡️ Pattern:

* Financial targeting

---

### **5. Download**

* `OperaSetup.exe`

➡️ Installed new browser

---

### **6. Missing Downloads**

➡️ Other malware not from Chrome

---

---

# **8. FULL ATTACK STORY (COMBINED)**

## **What Happened**

1. **System exposed via RDP**
2. Attacker:

   * Brute forced / logged in
3. Used:

   * `accounting` account
4. Installed:

   * Firefox, Opera
5. Executed:

   * Suspicious binaries
6. Created:

   * Wallet files
7. Accessed:

   * Wallet data
8. Used browsers:

   * To access banks
9. Likely:

   * Financial fraud activity

---

# **9. KEY INVESTIGATION PRINCIPLES**

## **Always Do**

* Convert time → UTC
* Filter aggressively
* Correlate artifacts

---

## **Core Questions**

* Who? → Logs
* What? → Process logs
* When? → MFT timeline
* Why? → Browser

---

## **Big Insight**

➡️ No single artifact tells the full story
➡️ **Correlation = truth**

---

# **FINAL TAKEAWAY**

* **Event Logs** → Access + execution
* **NTFS/MFT** → File activity
* **Browser Artifacts** → Intent

---


